Both Comp and ET (entity type) have a few features removed and have left derivation out as it is long and not pretty, added here as reference for rest of system.  
Both of Comp_ID and ET_ID are enums, CompInfo and ETInfo are structs containing basic informations that is used to generate <code>Comp</code> and ET.
## Comp
```c++
template<Comp_ID id, typename ComponentType = typename CompInfo<id>::type>
struct Comp
{
	using type = ComponentType;
	static constexpr Comp_ID sortedBy = CompInfo<id>::sortedBy;
	static constexpr auto sparse = getCompSparse<id>();
	static constexpr int noOfETsWithComp = sparse[ET_ID::MAX_ET_ID];
	//array of ET_IDs which contain this component - mainly useful for testing
	static constexpr auto ETsWithComp = isInSparse<id, noOfETsWithComp>();
	static constexpr int sortGroup = positionalArray(sortArray(), uniqueElements<noOfUniqueElements(sortArray())>(sortArray()))[id];
};
```
## ET
```c++
template<ET_ID id>
struct ET
{
	//which ETs are an ET<id> (not just direct inheritors). 
	static constexpr int noOfInheritors = noOfUniqueElements(getInheritors<id>::value);
	static constexpr std::array<ET_ID, noOfInheritors> inheritors = uniqueElements<noOfInheritors>(getInheritors<id>::value);
	//inclusive inheritors - same as inheritors but includes id, usefull for loops ect
	static constexpr std::array<ET_ID, noOfInheritors+1> incInheritors = concatinate(id,inheritors);
	//Components
	static constexpr int noOfComponents = noOfUniqueElements(concatinate(getComponents<id>::value, ETInfo<id>::newComponents));
	static constexpr std::array<Comp_ID, noOfComponents> components =
		uniqueElements<noOfComponents>(concatinate(getComponents<id>::value, ETInfo<id>::newComponents));
	//Sparse for getting order of components - used in ETData for ease of use
	static constexpr std::array<int, MAX_COMP_ID> sparse = CompSparse(components);
};
```
# ETData
```c++
#include "comp.hpp"

template<ET_ID id, int compIndex = 0, int lastComp = ET<id>::noOfComponents - 1> //-1 for easier specialization
struct ETDataTupleConstructor
{
	using CompType = Comp<ET<id>::components[compIndex]>::type;
	static constexpr auto data = std::tuple_cat(std::make_tuple(CompType()), ETDataTupleConstructor<id, compIndex + 1, lastComp>::data);
};

template<ET_ID id, int compIndex>
struct ETDataTupleConstructor<id, compIndex, compIndex>
{
	using CompType = Comp<ET<id>::components[compIndex]>::type;
	static constexpr std::tuple<CompType> data = {};
};

template<ET_ID id>
struct ETData
{
	using type = std::remove_const<decltype(ETDataTupleConstructor<id>::data)>::type;
	type data;
	//essentially should be std::get but with comp_id -> component position in ET mapping
	template<Comp_ID comp_id>
	constexpr Comp<comp_id>::type& get()
	{
		static_assert(ET<id>::sparse[comp_id] != Comp_ID::MAX_COMP_ID);
		return std::get<ET<id>::sparse[comp_id]>(data);	
	}
	template<Comp_ID comp_id>
	constexpr Comp<comp_id>::type&& move()//is move here ok or bad? not clear when using std::get on a member of class.
	{
		static_assert(ET<id>::sparse[comp_id] != Comp_ID::MAX_COMP_ID);
		return std::move(std::get<ET<id>::sparse[comp_id]>(data));
	}
};
```
# Entity32Bit
```c++
static constexpr uint32_t maxEntityType = 0xFFF;
static constexpr uint32_t maxEntityNumber = 0xFFFFF;
static constexpr uint32_t entityValueBits = 20;

//options with this set up: max 1m entities, 4095 different entity types
class Entity32Bit
{
private:
	uint32_t mEntity;
public:
	constexpr uint32_t number() const  noexcept
	{
		return mEntity & maxEntityNumber;
	}
	constexpr uint32_t type() const noexcept
	{
		return (mEntity >> entityValueBits);// &maxEntityType);
	}
	constexpr void addType(uint32_t type) noexcept
	{
		assert(type <= maxEntityType);
		mEntity |= (type << entityValueBits);
	}
	constexpr void addNumber(const uint32_t entityNum) noexcept
	{
		assert(entityNum <= maxEntityNumber);
		mEntity = entityNum + (this->type() << entityValueBits);
	}
	inline bool operator==(const Entity32Bit rhs) const noexcept
	{
		return this->number() == rhs.number() && this->type() == rhs.type();
	}

	Entity32Bit() noexcept :mEntity(0) {}
	constexpr Entity32Bit(const uint32_t entityNumber, const uint32_t type) noexcept : mEntity(entityNumber)
	{
		assert(entityNumber < maxEntityNumber);
		addType(type);
	}
};
```
# Sparse Set, sorting has been removed as it would make this post to long

```c++
#include <vector>
#include "Entity.hpp"
#include "Comp.hpp"

class SegSparseSet
{
private:
	std::array<std::vector<Entity32Bit>, MAX_ET_ID> mEDS; //Entity Dense Sets
	std::array<std::vector<uint32_t>, MAX_ET_ID> mSparses;
public:
	inline bool entityInSet(Entity32Bit entity) noexcept { return (mSparses[entity.type()][entity.number()] != _UI32_MAX); }

	inline std::vector<Entity32Bit>& getEntities(const uint32_t group) { return mEDS[group]; }
	inline Entity32Bit& getEntity(const ET_ID group, const uint32_t index) { return mEDS[group][index]; }
	inline uint32_t getIndex(const Entity32Bit entity) { return mSparses[entity.type()][entity.number()]; }
	inline void changeIndex(const Entity32Bit entity, const uint32_t value) { mSparses[entity.type()][entity.number()] = value; }

	void addEntity(const Entity32Bit entity)
	{
		assert(!entityInSet(entity));
		changeIndex(entity, mEDS[entity.type()].size());
		mEDS[entity.type()].push_back(entity);
	}
	void deleteEntity(const Entity32Bit entity)
	{
		assert(entityInSet(entity));
		//change last member in group to point to deleted component;
		changeIndex(*(mEDS[entity.type()].end() - 1), getIndex(entity));
		//swapComponent + delete EDS
		mEDS[entity.type()][getIndex(entity)] = *(mEDS[entity.type()].end() - 1);
		mEDS[entity.type()].pop_back();
		//clear entity in sparse
		changeIndex(entity, _UI32_MAX);
	} 
	uint32_t totalSize()
	{
		int size = mEDS[1].size(); //mEDS[0] is always empty
		for (int i = 2; i < MAX_ET_ID; ++i)
		{
			size += mEDS[i].size();
		}
		return size;
	}
	void resizeSparse(ET_ID id, uint32_t size)
	{
		mSparses[id].resize(size);
		for (int i = 0; i < size; ++i)
		{
			mSparses[id][i] = _UI32_MAX;
		}
	}  
	uint32_t size(ET_ID id)
	{
		return mEDS[id].size();
	}

	SegSparseSet() noexcept :mSparses()
	{
		for (int i = 0; i < MAX_ET_ID; ++i)
		{
			resizeSparse((ET_ID)i, maxEntityAmount()[i]);
		}
	}
	template<Comp_ID component>
	SegSparseSet(const Comp<component>& comp) noexcept :mSparses()
	{
		for (int i = 0; i < MAX_ET_ID; ++i)
		{
			if (Comp<component>::compArray[i] == true)
			{
				resizeSparse((ET_ID)i, maxEntityAmount()[i]);
			}
		}
	}
};
  
template<Comp_ID mID, typename CompType = typename Comp<mID>::type>
class TypeSortedSS
{
private:
	using component = Comp<mID>;
	std::array<std::vector<CompType>, MAX_ET_ID> mCDS; //component dense set, parallel to mEDS in segSS.
	SegSparseSet* mpSS;
public:
  //checks to see if entity has this component
	inline bool validEntityGroup(Entity32Bit entity) noexcept { return (component::sparse[entity.type()] != 0); }
  
	void addComponent(Entity32Bit entity, const CompType& data)
	{
		assert(validEntityGroup(entity));
		mCDS[entity.type()].push_back(data);
	}
	void deleteComponent(Entity32Bit entity)
	{
		//need to do this check here (atleast in debug) as entity in SharedSS is deleted after components
		assert(mpSS->entityInSet(entity) && validEntityGroup(entity));
		mCDS[entity.type()][mpSS->getIndex(entity)] = *(mCDS[entity.type()].end() - 1);
		mCDS[entity.type()].pop_back();
	}
public:
	inline auto end(ET_ID id) { return mCDS[id].end(); }
	inline auto begin(ET_ID id) { return mCDS[id].begin(); }

	inline uint32_t getNoOfET(ET_ID id) { return mCDS[id].size(); }
	inline std::vector<CompType>& getETComps(ET_ID id) { return mCDS[id]; }
	inline CompType& getComponent(Entity32Bit entity) { return mCDS[entity.type()][mpSS->getIndex(entity)]; }
	inline CompType& getComponent(uint32_t eType, uint32_t index) { return mCDS[eType][index]; }
	inline void addSegmentedSS(SegSparseSet* SS) { mpSS = SS; }
	inline Entity32Bit getEntity(uint32_t eType, uint32_t index)  { return mpSS->getEntity(eType, index); }
  
	TypeSortedSS() : mpSS(nullptr) {}
};
```
Does it make more sense to move all of TypeSortedSS into EntityManager, and replace mSparses with a tuple of mCDS?
# EntityManager

```c++
#include "ETData.hpp"
#include "TypeSortedSS.hpp"

template<int... ints>
constexpr auto genTypesForTypeSortedTuple(std::integer_sequence<int, 0, ints...> seq)
{
	return std::tuple<int, TypeSortedSS<(Comp_ID)ints>...>();
}
typedef decltype(genTypesForTypeSortedTuple(std::make_integer_sequence<int, MAX_COMP_ID>())) TypeSortedSSTuple;

class EntityManager
{
private:
	//size of array == number of sorting groups
	std::array<SegSparseSet,noOfUniqueElements(sortArray())> mSharedSSs;
	TypeSortedSSTuple mSparses;
	std::array<uint32_t,MAX_ET_ID> mNextEntityNum;
	std::array<std::vector<uint32_t>, MAX_ET_ID> mDeletedEntityNum;
public:
	template<Comp_ID component>
	inline auto& sparse() { return std::get<component>(mSparses); } //for testing

	template<ET_ID id>
	Entity32Bit addEntity(ETData<id>& data)
	{
		Entity32Bit entity;
		if (mDeletedEntityNum[id].size() == 0)
		{
			assert(mNextEntityNum[id] < maxEntityNumber);
			entity.addNumber(mNextEntityNum[id]++);
			entity.addType(id);
		}
		else
		{
			entity.addNumber(*(mDeletedEntityNum[id].end() - 1));
			entity.addType(id);
			mDeletedEntityNum[id].pop_back();
		}
		//this makes assumption that at least one component of each entity is unsorted.
		mSharedSSs[0].addEntity(entity);
		addData(entity, data);
		return entity;
	}
	
	template<ET_ID id>
	void deleteEntity(Entity32Bit entity)
	{
		removeData<id>(entity);
		mSharedSSs[0].deleteEntity(entity);
		mDeletedEntityNum[id].push_back(entity.number());
	}
private:
	template<ET_ID id, int index = ET<id>::noOfComponents - 1>
	void addData(Entity32Bit entity, ETData<id>& data)
	{
		//if sorted by itself add entity to the correct sorted SharedSS
		if constexpr (Comp<ET<id>::components[index]>::sortedBy == ET<id>::components[index])
		{
			mSharedSSs[Comp<ET<id>::components[index]>::sortGroup].addEntity(entity);
		}
		std::get<ET<id>::components[index]>(mSparses).addComponent(entity, data.get<ET<id>::components[index]>());
		if constexpr (index != 0)
		{
			addData<id, index - 1>(entity, data);
		}
	}

	template<ET_ID id, int index = ET<id>::noOfComponents - 1>
	void removeData(Entity32Bit entity)
	{
		std::get<ET<id>::components[index]>(mSparses).deleteComponent(entity);
		//if sorted by itself delete entity from the correct sorted SharedSS
		if constexpr (Comp<ET<id>::components[index]>::sortedBy == ET<id>::components[index])
		{
			mSharedSSs[Comp<ET<id>::components[index]>::sortGroup].deleteEntity(entity);
		}
		if constexpr (index != 0)
		{
			removeData<id, index - 1>(entity);
		}
	}
public:
	//both size functions assume at least one component of each ET is unsorted.
	inline int size() { return mSharedSSs[0].totalSize(); }
	inline uint32_t noOfET(ET_ID id) { return mSharedSSs[0].size(id); }
	
	//returns an iterator for dense set of the component for ET<id>
	template<Comp_ID component>
	inline auto begin(ET_ID id) { return std::get<component>(mSparses).begin(id); }
	template<Comp_ID component> 
	inline auto end(ET_ID id) { return std::get<component>(mSparses).end(id); }

	//looks simular to calling by Entity32Bit, however it bypasses looking through the sparse to get index, so is faster if you know index.
	template<Comp_ID component, typename ReturnType = typename Comp<component>::type>
	inline ReturnType& getComp(ET_ID id, uint32_t index) { return std::get<component>(mSparses).getComponent(id, index); }
	template<Comp_ID component, typename ReturnType = typename Comp<component>::type>
	inline ReturnType& getComp(Entity32Bit entity) { return std::get<component>(mSparses).getComponent(entity); }
	template<Comp_ID component>
	inline Entity32Bit getEntity(ET_ID id, uint32_t index) { return mSharedSSs[Comp<component>::sortGroup].getEntity(id, index); }
private:
	template<int index = 1>
	void addSegmentedSS()
	{
		if constexpr (index < MAX_COMP_ID)
		{
			std::get<index>(mSparses).addSegmentedSS(&mSharedSSs[Comp<(Comp_ID)index>::sortGroup]);
			addSegmentedSS<index + 1>();
		}
		return;
	}
public:
	EntityManager() noexcept : mSharedSSs()
	{
		for (int i = 0; i < MAX_ET_ID; ++i)
		{
			mNextEntityNum[i] = 1;
		}
		addSegmentedSS();
	};
};
Does it make more sense to move all of TypeSortedSS into EntityManager, and replace mSparses with a tuple of mCDS?
```

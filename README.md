# Thinc CoreData
[![Build Status](https://api.travis-ci.org/webair/thinc.swift.THCCoreData.svg)](https://travis-ci.org/webair/thinc.swift.THCCoreData)

A Core Data Extension written with Swift. 

# Usage
Make sure that all of you managed objects implements the protocol 'NamedManageObject' and return its entity name.

You can also use [mogenerator](https://github.com/rentzsch/mogenerator) to generate this behaviour (Very nice tool, best thanks to the creators!). If using the mogenerator you need to add a base class which implements the 'ManagedObjectEntity' protocol or use my [fork](https://github.com/webair/mogenerator) which allowes you to add a protocol parameter and framework includes to the swift template:
    
    mogenerator --base-class-import "THCCoreData" \
                --protocol "ManagedObjectEntity" \
                ...

## ContextManager
Get the default manager. It will create a sqlite store in the documents folder and merges all object model files from the main bundle.  

    let manager = ContextManager()
    let mainContext = manager.mainContext
        
Alternativly you can also initilize a customized context manager by passing a managedObjectModel (the sqlite store will still get created for you)...
	
	let manager = let manager = ContextManager(managedObjectModel: customModel)
	
... or by giving the complete persistant store coordinator

	let manager = ContextManager(persistentStoreCoordinator: customCoordinator)
	
When using the sqlite auto create initializers, you can set the 'recreateStoreIfNeeded' to true, this will retry creating the sqlite store if an error occures, by deleting the store first. This should only be used during development, because it can lead to data loss!

	let manager = ContextManager(recreateStoreIfNeeded: true)

	
If you like to work with a singleton manager you can easily create your own ContextManager extension:

	public extension ContextManager {
    	static let defaultManager: ContextManager = {
        	return ContextManager()
    	}()
	}

If you need a new private context, you can get it from the manager:
    
    let privateContext = manager.privateContext
	
## NSManagedContext
This library extends the default NSManagedObjectContext class. Assume we have generated a managed object subclass 'MyObject' from an entity, you can now insert it to the context like that:

    let context = ContextManager.defaultManager.mainContext
    let myObject = context.createObject(MyObject)
    
You can also get a default NSFetchRequest for a given managed object subclass:
    
    let fetchRequest = context.fetchRequest(MyObject)
    
Last but no least you can also get a RequestSet (see RequestSet) class from the context:

	let requestSet = context.requestSet(MyObject)
 
## RequestSet

This class was inspired by the django framework for python (QuerySet). It helps create a NSFetchRequest and let you easily create fetch requests. It will aggregate the request settings and only perform a fetch as soon as the data is accessed.


### init request set

	let requestSet = RequestSet<MyManagedObject>(context: manager.mainContext)

### Accessing data
	
	// get the current count (this wont yet execute the fetch)
	let count = requestSet.count
	
	// iterate over result set
	for obj in requestSet {
		println(obj.name)
	}
	// access elements directly
	println(requestSet[0])

### Filters
    
    // filter with predicate
    requestSet.filter(NSPredicate(format:"name='test'"))
    
    // filter with value
    requestSet.filter("name", value:"test")
    
    // filter with tuples (default AND)
    requestSet.filter([("name", "test"), ("name", "test2")])
    
    // OR filter with tuples
    requestSet.filter([("name", "test"), ("name", "test2")], mode: RequestFilterMode.OR)
    
    // chaining filters ...
    requestSet.filter(NSPredicate(format:"name='test'"))
        .filter(NSPredicate(format:"name='test2'"))
    // ... with OR
    requestSet.filter(NSPredicate(format:"name='test'"))
        .filter(NSPredicate(format:"name='test2'"), mode: RequestFilterMode.OR)
    
    // mixing filters (will create name='test' OR name='test2' AND name='test3')
    requestSet.filter([("name", "test"), ("name", "test2")], mode: RequestFilterMode.OR)
        .filter("name", value: "test3")
    
    // and so on ...
	
### Limit

	// will limit the request set to then
	requestSet.limit(3)
	
	
### Sorting

    // sort by 'name' (default ascending)
    requestSet.sortBy("name")
    
    // sort desecending
    requestSet.sortBy("name", order:RequestSortOrder.DESCENDING)
    
    // multiple sortings (will sort first the attribute 'name' and secondary the attribute 'otherAttribute')
    requestSet.sortBy([
        ("name", RequestSortOrder.DESCENDING),
        ("otherAttrinute",RequestSortOrder.ASCENDING)]
    )
    
### All together

Because of the chaining it is possible to produce following fetch request:
 
    // will filter all entries with name='test' or firstname='test' sorted by name
    // with descending order and limit the result to three
    requestSet.filter("name", value:"test")
        .filter("firstname", value:"test", mode: RequestFilterMode.OR)
        .sortBy("name", order: RequestSortOrder.DESCENDING)
        .limit(3)
	
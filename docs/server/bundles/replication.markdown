##Replication

Raven replication can be enabled by dropping the Raven.Bundles.Replication.dll to Raven's Plugins directory.

You can read about potential deployment options for the replication bundle here: [Mixing Replication and Sharding](http://ravendb.net/docs/server/bundles/replicationandsharding). 
The replication module will effect the following changes:

* Track the server the document was originally written on. The replication bundle uses this information to determine if a replicated document is conflicting with the existing document.
* Documents that encountered a conflict would be marked appropriately and will require automated or user involvement to resolve.
* Document deletes result in delete markers, which the replication bundle needs in order to be able to replicate deletes to sibling instances. This is an implementation detail and is not noticeable to clients.
* Several new endpoints will begin responding, including, but not limited to:
* * /replication/replicate
* * /replication/lastEtag
* The replication bundle will create several system documents, including, but not limited to:
* * Raven/Replication/Destinations - List of servers we need to replicate to
* * Raven/Replication/Sources/[server] - Information about the data replicated from a particular server
* The replication bundle will not replicate any system documents (whose key starts in Raven/)

##Format of Raven/Replication/Destinations
The destination document format is:

    {  
      "Destinations": [  
        {  
          "Url": "http://raven_two:8080/"  
        },  
        {  
          "Url": "http://raven_three:8080/"  
        },  
      ]
    }

With an object containing a url per each instance to replicate to. Whenever this document is updated, replication kicks off and start replicating to the updates destination list.

## How replication works?
On every transaction commit, Raven will look up the list of replication destination. For each of the destination, the replication bundle will:

* Query the remote instance for the last document that we replicated to that instance.
* Start sending batches of updates that happened since the last replication.

Replication happens in the background and in parallel. 

##What about failures?
Because of the way it is designed, a node can fail for a long period of time, and still come up and start accepting everything that it missed in the meanwhile. The Replication Bundle keeps track of only a single data item per each replicating server, the last etag seen from that server. This means that we don't have to worry about missing replication windows.

When the replication bundle encountered a failure when replicating to a server, it has an intelligent error handling strategy. It is meant to be hands off, so nodes can fail and come back up without any administrative intervention. The strategy is outlined below:

* If this is the first failure encountered, immediately try to replicate the same information again. This is done under the assumption that most failures are transient in nature.
* If the second attempt fails as well, we note the failure (in Raven/Replication/Destinations/[server url])
* After ten consecutive failures, Raven will start replicating to this node less often
* * Once every 10 replication cycles, until failure count reaches 100
* * Once every 100 replication cycles, until failure count reaches 1,000
* * Once every 1,000 replication cycles, when failure count is above 1,000
* Any successful replication will reset the failure count, on the assumption that

##Can I bring up a new node and start replicating to it?
Yes, you can. You can edit the Raven/Replication/Destinations document in the replicating instance to add the new node, and the Replication Bundle will immediately start replicating to that server.

##What about timeouts?
By default, the replication bundle have a timeout of 500 ms. You can control that by specifying "Raven/Replication/ReplicationRequestTimeout" in the configuration file <appSettings/> section.

The default value assumes servers that are near one another, for replication over the WAN, you would likely want to add additional time. Please note that timeout failures for the replication bundle counts as failures, and the replication bundle will reduce the number of replication attempts against a node that fails often.

## What happen if there is a conflict?

In a replicating system, it is possible that two writes to the same document will occur on two different servers, resulting in two independent versions of the same document. When replication occur between these two version, the Replication Bundle is faced with a problem. It has two authentic versions of the same thing, saying different things. At that point, the Replication Bundle will mark that document as conflicting, store all the conflicting documents in a safe place and set the document content to point to the conflicting documents.

Resolving a conflict is easy, you just need to PUT a new version of the document. On PUT, the Replication Bundle will consider the conflict resolved.

More details about conflicts are here: [Dealing with replication conflicts](http://ravendb.net/docs/server/bundles/replicationconflicts).

##Client integration
Raven's Client API will detect and respond appropriately whenever a server has the replication bundle installed. This includes:

* Detecting that an instance is replicating to another set of instances.
* When that instance is down, will automatically shift to the other instances.

The Raven Client API is quite intelligent in this regard, upon failure, it will:

* Assume that the failure is transient, and retry the request.
* If the second attempt fails as well, we record the failure and shift to a replicated node, if available.
* After ten consecutive failures, Raven will start replicating to this node less often
* * Once every 10 requests, until failure count reaches 100
* * Once every 100 requests, until failure count reaches 1,000
* * Once every 1,000 requests, when failure count is above 1,000
* On the first successful request, the failure count is reset.

If the second replicated node fails, the same logic applies to it as well, and we move to the third replicated node, and so on. If all nodes fail, an appropriate exception is thrown.

At a lower level, those are the operations that support replication:

* Get - single document and multi documents
* Put
* Delete
* Query
* Rollback
* Commit

The following operation do not support replication in the Client API:

* PutIndex
* DeleteIndex
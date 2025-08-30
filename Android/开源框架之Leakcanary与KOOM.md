# Leakcanary

```kotlin
private fun updateTrie(  
  pathNode: ReferencePathNode,  
  path: List<Long>,  
  pathIndex: Int,  
  parentNode: ParentNode  
) {  
  val objectId = path[pathIndex]  
  if (pathIndex == path.lastIndex) {  
    parentNode.children[objectId] = LeafNode(objectId, pathNode)  
  } else {  
    val childNode = parentNode.children[objectId] ?: run {  
      val newChildNode = ParentNode(objectId)  
      parentNode.children[objectId] = newChildNode  
      newChildNode  
    }  
    if (childNode is ParentNode) {  
      updateTrie(pathNode, path, pathIndex + 1, childNode)  
    }  
  }  
}
```


```kotlin
private fun deduplicateShortestPaths(  
  inputPathResults: List<ReferencePathNode>  
): List<ShortestPath> {  
  val rootTrieNode = ParentNode(0)  
  
  inputPathResults.forEach { pathNode ->  
    // Go through the linked list of nodes and build the reverse list of instances from  
    // root to leaking.    val path = mutableListOf<Long>()  
    var leakNode: ReferencePathNode = pathNode  
    while (leakNode is ChildNode) {  
      path.add(0, leakNode.objectId)  
      leakNode = leakNode.parent  
    }  
    path.add(0, leakNode.objectId)  
    updateTrie(pathNode, path, 0, rootTrieNode)  
  }  
  
  val outputPathResults = mutableListOf<ReferencePathNode>()  
  findResultsInTrie(rootTrieNode, outputPathResults)  
  
  if (outputPathResults.size != inputPathResults.size) {  
    SharkLog.d {  
      "Found ${inputPathResults.size} paths to retained objects," +  
        " down to ${outputPathResults.size} after removing duplicated paths"    }  
  } else {  
    SharkLog.d { "Found ${outputPathResults.size} paths to retained objects" }  
  }  
  
  return outputPathResults.map { retainedObjectNode ->  
    val shortestChildPath = mutableListOf<ChildNode>()  
    var node = retainedObjectNode  
    while (node is ChildNode) {  
      shortestChildPath.add(0, node)  
      node = node.parent  
    }  
    val rootNode = node as RootNode  
    ShortestPath(rootNode, shortestChildPath)  
  }  
}
```
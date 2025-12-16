# Vector Databases

for what:
- speed search in unstructured data ( image, video, text, sound )
- find similarities vs non-similarities

## The Problem They Solve

**Traditional databases** search for exact matches:
- "Find all users named 'John'" ✓
- "Find products with price < $50" ✓
- "Find records where status = 'active'" ✓

**But they struggle with similarity and meaning:**
- "Find images similar to this one" ✗
- "Find documents about 'AI' even if they don't contain that exact word" ✗
- "Recommend products based on user preferences" ✗

## What Vector Databases Do

Vector databases store and search **embeddings** (numerical representations of data) to find **semantic similarity**.
The primary purpose of vector databases is to provide fast and accurate similarity search or nearest neighbor search capabilities.  
The integration of AI techniques in vector databases enhances their capabilities, improves search accuracy, optimizes performance, and enables more intelligent and efficient management of high-dimensional data.

### Example
```python
# Text gets converted to vectors (embeddings)
"dog" → [0.2, 0.8, 0.1, ...]
"puppy" → [0.25, 0.75, 0.15, ...]  # Similar vector!
"car" → [0.9, 0.1, 0.3, ...]       # Very different vector

# Vector DB can find that "dog" and "puppy" are similar
# even though they're different words
```

## Scalability  
Vector databases are optimized for handling tens of millions to billions of data objects, enabling real-time searches even with large datasets.

Vector databases use a class of algorithm called ANN to find which vectors are 
in the neighboorhood of a query vector.

- Approximate Nearest Neighbors (ANN): This algorithm allows efficient searching by finding vectors close to the query vector without comparing every stored vector, trading some accuracy for speed.

- Hierarchical Navigable Small Worlds (HNSW): A popular ANN algorithm that builds a graph of vectors, enabling efficient searches by navigating through different levels of the graph to find the closest neighbors.

## Ways to measure the performance of a vector DB
These metrics help assess the performance and efficiency of vector databases.

- Recall: Measures how accurately the database retrieves the nearest neighbors. Higher recall means better accuracy. How reliable it returns the correct nearest neighbors.
- Queries Per Second (QPS): Indicates the speed of the database, showing how many queries can be processed per second. Higher QPS means better performance. How fast is it.
- Memory Usage: Refers to the amount of RAM required to run the database. Lower memory usage is ideal as it reduces costs. How much memory it costs to run the queries.

The trade off is between QPS and recall.

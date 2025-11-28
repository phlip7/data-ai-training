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


# Bloom Filter

A **Bloom filter** is a space-efficient probabilistic data structure, conceived by Burton Howard Bloom in 1970, that is used to test whether an element is a member of a set.

## Why use a Bloom Filter?
When you have a massive dataset (like millions of registered usernames), querying the database to check if a username is available can be extremely slow and resource-intensive. A Bloom Filter acts as a rapid, in-memory filter in front of your database.

### Key Characteristics:
- **False positives are possible**: It might say *yes, this element is in the set* when it actually isn't.
- **False negatives are impossible**: It will never say *no, this element is NOT in the set* if it actually is.
- **Extremely space-efficient**: Requires very few bits per element.

## How it Works
1. **The Bit Array**: An array of `m` bits, all initialized to `0`.
2. **Hash Functions**: `k` different hash functions, each mapping an item to one of the `m` array positions.
3. **Insertion**: To add an item, feed it to each of the `k` hash functions to get `k` array positions. Set the bits at all these positions to `1`.
4. **Lookup**: To check if an item is in the set, feed it to each of the `k` hash functions to get `k` array positions.
   - If *any* of the bits at these positions is `0`, the item is **definitely not** in the set.
   - If *all* bits are `1`, then either the item is in the set, or the bits have by chance been set to `1` during the insertion of other elements (resulting in a false positive).

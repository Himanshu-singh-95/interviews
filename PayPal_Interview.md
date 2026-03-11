# Complete Hand (PayPal interview question)

## Problem

You're creating a simplified Mahjong-like game. Tiles are digits `0`–`9`. Tiles can be grouped into:
- triples (three of the same digit), and
- exactly one pair (two of the same digit).

A "complete hand" is a multiset of tiles where every tile is used exactly once, the grouping contains zero or more triples, and exactly one pair.

Write a function that takes a string of tiles (digits) in arbitrary order and returns `true` or `false` depending on whether the collection represents a complete hand.

## Examples

- `"88844"` => true (triple 8, pair 4)
- `"99"` => true (just a pair)
- `"55555"` => true (triple 5 and pair 5)
- `"22333333"` => true (pair 2, two triples of 3)
- `"111333555"` => false (three triples, no pair)
- `"42"` => false (two different singles)
- Other test cases (from the original problem) include longer strings; see `tiles_completehand.js` for full list.

## Constraints

- N = number of tiles (length of input string).
- Tiles are characters `'0'`..`'9'`.

## Key idea / Algorithm

1. Build a frequency map of digits (counts for each tile).
2. A valid complete hand must have exactly one chosen digit used as the pair (count >= 2). For each candidate digit that can be the pair:
   - Subtract two from its frequency.
   - Check that every remaining frequency is divisible by 3 (so they can form triples).
   - If yes, the hand is complete.
3. If no candidate pair leads to all counts being multiples of 3, the hand is not complete.

This is O(N + D) time where D = 10 (number of distinct tile types), so effectively O(N). Space is O(D).

## JavaScript implementation

The implementation in the repository was updated to a clearer variant that avoids in-place mutation and adds a fast length check. Use this version for readability and maintainability:

```javascript
function complete(tiles) {
  // Quick length check: hand must be pair + zero or more triples => length % 3 === 2
  if (tiles.length % 3 !== 2) return false;

  const counts = {};
  for (const tile of tiles) {
    counts[tile] = (counts[tile] || 0) + 1;
  }

  // Try using each distinct tile as the "exactly one pair"
  for (const tile of Object.keys(counts)) {
    if (counts[tile] >= 2) {
      const remaining = { ...counts };
      remaining[tile] -= 2;

      // Check if all remaining tiles can form triples
      const canFormTriples = Object.values(remaining).every(c => c % 3 === 0);
      if (canFormTriples) return true;
    }
  }

  return false;
}
```

A small runner has been added to `tiles_completehand.js` that prints the example results when run with Node.

## Complexity

- Time: O(N) to build frequencies; checking candidates iterates over at most 10 keys each time, so O(N + 10 * 10) ≈ O(N).
- Space: O(10) = O(1) auxiliary.

## Try it

If you have the implementation in a file `tiles_completehand.js`, run:

```bash
node tiles_completehand.js
```

(or create a small runner that imports/executes `complete(...)`).

## Notes

- The approach assumes only triples and exactly one pair are allowed. It does not try to form sequences (unlike full Mahjong).
- The implementation is deterministic and efficient because the alphabet size is constant (10 digits).

---

File referenced: `tiles_completehand.js` (contains example inputs and the `complete` function).

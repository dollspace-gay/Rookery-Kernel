# Tier-3: lib/rbtree.c — Red-Black tree

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: lib/00-overview.md
upstream-paths:
  - lib/rbtree.c (~618 lines)
  - include/linux/rbtree.h
  - include/linux/rbtree_types.h
  - include/linux/rbtree_augmented.h
  - Documentation/core-api/rbtree.rst
-->

## Summary

The **Red-Black tree** is the kernel's intrusive self-balancing binary search tree. Per-node lives `struct rb_node` (parent + color packed in low bit, left, right) embedded in the user's containing struct. Per-tree anchor is `struct rb_root` (root pointer). Per-cached variant `struct rb_root_cached` adds a per-leftmost cache for O(1) `rb_first`. Per-linked variant `struct rb_root_linked` threads a doubly-linked list through nodes for O(1) iteration. Per-coloring discipline: root black; red nodes have black children; every root-to-NULL path has equal black-node count. Per-`rb_link_node` + `rb_insert_color`: caller does the BST descent then library rebalances. Per-`rb_erase`: library detaches node and rebalances. Per-`rb_next` / `_prev`: in-order successor / predecessor. Per-`rb_entry`: `container_of` wrapper. Used pervasively: mm VMA-tree (per-process), CFS / EEVDF scheduler run-queue, FIB tries, ext4 / btrfs extents, epoll, /proc, fcntl posix-locks, kprobes, perf-events, fs-notify marks, and many more.

This Tier-3 covers `lib/rbtree.c` (~618 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rb_node` | per-node (parent+color, left, right) | `RbNode` |
| `struct rb_root` | per-tree (root pointer) | `RbRoot` |
| `struct rb_root_cached` | per-tree + leftmost-cache | `RbRootCached` |
| `struct rb_node_linked` | per-node with prev/next list | `RbNodeLinked` |
| `struct rb_root_linked` | per-tree + linked-leftmost | `RbRootLinked` |
| `struct rb_augment_callbacks` | per-augment-tree callbacks | `RbAugmentCallbacks` |
| `RB_ROOT` | per-empty-tree init | `RbRoot::EMPTY` |
| `RB_ROOT_CACHED` | per-empty-cached init | `RbRootCached::EMPTY` |
| `RB_ROOT_LINKED` | per-empty-linked init | `RbRootLinked::EMPTY` |
| `rb_entry(ptr, type, member)` | per-container_of | `RbNode::entry!` macro |
| `rb_entry_safe(ptr, type, member)` | per-null-safe | `RbNode::entry_safe!` macro |
| `rb_link_node()` (inline) | per-attach new node at descent leaf | `RbNode::link` |
| `rb_link_node_rcu()` (inline) | per-attach RCU-safe | `RbNode::link_rcu` |
| `rb_insert_color()` | per-rebalance after BST-insert | `RbTree::insert_color` |
| `rb_insert_color_cached()` (inline) | per-insert + update leftmost | `RbTreeCached::insert_color` |
| `__rb_insert_augmented()` | per-augment-aware insert | `RbTree::insert_augmented` |
| `rb_erase()` | per-detach + rebalance | `RbTree::erase` |
| `rb_erase_cached()` (inline) | per-erase + update leftmost | `RbTreeCached::erase` |
| `rb_erase_linked()` | per-erase + list unlink | `RbTreeLinked::erase` |
| `__rb_erase_color()` | per-rebalance helper (exposed for augmented) | `RbTree::erase_color` |
| `rb_first()` (inline) | per-leftmost (in-order first) | `RbTree::first` |
| `rb_last()` (inline) | per-rightmost (in-order last) | `RbTree::last` |
| `rb_next()` | per-in-order successor | `RbTree::next` |
| `rb_prev()` | per-in-order predecessor | `RbTree::prev` |
| `rb_replace_node()` | per-substitute node | `RbTree::replace_node` |
| `rb_replace_node_rcu()` | per-substitute RCU-safe | `RbTree::replace_node_rcu` |
| `rb_first_postorder()` | per-postorder first (for destroy) | `RbTree::first_postorder` |
| `rb_next_postorder()` | per-postorder next | `RbTree::next_postorder` |
| `rbtree_postorder_for_each_entry_safe()` | per-destroy iteration macro | `rb_postorder_iter!` |

## Compatibility contract

REQ-1: struct rb_node:
- __rb_parent_color: per-`unsigned long` containing parent pointer in high bits + color in low bit (RB_RED=0, RB_BLACK=1).
- rb_right: per-`struct rb_node *` (NULL = sentinel-NIL).
- rb_left: per-`struct rb_node *` (NULL = sentinel-NIL).
- Per-alignment: rb_node must be at-least 4-byte aligned (low bit reserved).

REQ-2: struct rb_root:
- rb_node: per-`struct rb_node *` root.
- RB_ROOT = { NULL, }.

REQ-3: struct rb_root_cached:
- rb_root: per-base rb_root.
- rb_leftmost: per-cached leftmost node (or NULL).
- RB_ROOT_CACHED = { {NULL,}, NULL }.

REQ-4: struct rb_node_linked / rb_root_linked:
- rb_node_linked embeds rb_node + prev/next pointers (doubly linked through in-order sequence).
- rb_root_linked: rb_root + rb_leftmost (linked-variant).

REQ-5: Color invariants (RB-tree properties):
- (I1) Every node is RED or BLACK.
- (I2) Root is BLACK.
- (I3) Every NULL (sentinel) is BLACK.
- (I4) If a node is RED, both children are BLACK (no two reds in a row).
- (I5) Every path from a node to descendant NULLs contains the same number of BLACK nodes.
- ⟹ Maximum depth ≤ 2·log2(N+1).

REQ-6: rb_link_node(node, parent, link) (inline):
- node.__rb_parent_color = (unsigned long) parent.   /* RED = 0 by default */
- node.rb_left = node.rb_right = NULL.
- *link = node.   /* link is &parent.rb_left or &parent.rb_right */

REQ-7: rb_insert_color(node, root):
- /* node has just been BST-attached as RED leaf */
- __rb_insert(node, root, dummy_rotate).
- Loop: while parent(node) is RED:
  - Determine grandparent gp; uncle u.
  - Case 1 (uncle RED): recolor parent + uncle BLACK, gp RED, node = gp, continue.
  - Case 2 (uncle BLACK, node is "outer"): rotate gp (single rotation); recolor; done.
  - Case 3 (uncle BLACK, node is "inner"): rotate parent; reduce to Case 2.
- Set root.rb_node BLACK (I2).

REQ-8: rb_erase(node, root):
- /* Standard BST delete with successor replacement */
- 3 cases on number of children: 0, 1, 2.
- Compute "rebalance" node (the node whose subtree lost a black level).
- __rb_erase_color(rebalance, root, dummy_rotate) if non-NULL.

REQ-9: __rb_erase_color(parent, root, augment_rotate):
- /* Restore I5 after losing one black */
- Loop: traverse upward.
- Case 1: sibling RED: rotate parent; recolor; reduce.
- Case 2: sibling BLACK with two BLACK children: recolor sibling RED; move up.
- Case 3: sibling BLACK with outer child BLACK, inner RED: rotate sibling; reduce to Case 4.
- Case 4: sibling BLACK with outer child RED: rotate parent; recolor; done.

REQ-10: rb_first(root) (inline):
- n = root.rb_node.
- if !n: return NULL.
- while n.rb_left: n = n.rb_left.
- return n.

REQ-11: rb_last(root) (inline):
- n = root.rb_node.
- if !n: return NULL.
- while n.rb_right: n = n.rb_right.
- return n.

REQ-12: rb_next(node):
- if RB_EMPTY_NODE(node): return NULL.
- if node.rb_right:
  - n = node.rb_right.
  - while n.rb_left: n = n.rb_left.
  - return n.
- /* Else walk up until we are a left child */
- while (parent = rb_parent(node)) ∧ node == parent.rb_right:
  - node = parent.
- return parent.

REQ-13: rb_prev(node):
- if RB_EMPTY_NODE(node): return NULL.
- if node.rb_left:
  - n = node.rb_left.
  - while n.rb_right: n = n.rb_right.
  - return n.
- while (parent = rb_parent(node)) ∧ node == parent.rb_left:
  - node = parent.
- return parent.

REQ-14: rb_replace_node(victim, new, root):
- *new = *victim.   /* copy parent/color/children */
- if rb_parent(victim):
  - if victim == rb_parent.rb_left: rb_parent.rb_left = new.
  - else: rb_parent.rb_right = new.
- else: root.rb_node = new.
- if victim.rb_left: rb_set_parent(victim.rb_left, new).
- if victim.rb_right: rb_set_parent(victim.rb_right, new).
- /* victim left in place but no longer reachable */

REQ-15: rb_replace_node_rcu(victim, new, root):
- /* RCU-safe variant: writes new pointers; readers observe atomically */
- new.__rb_parent_color = victim.__rb_parent_color.
- rcu_assign_pointer(new.rb_left, victim.rb_left).
- rcu_assign_pointer(new.rb_right, victim.rb_right).
- /* Update parent's link */
- if rb_parent(victim):
  - rcu_assign_pointer(rb_parent.rb_left-or-right, new).
- else: rcu_assign_pointer(root.rb_node, new).
- /* Children's parent pointer is not RCU-visible to lookups */
- if victim.rb_left: rb_set_parent(victim.rb_left, new).
- if victim.rb_right: rb_set_parent(victim.rb_right, new).

REQ-16: Postorder iteration (for destroy):
- rb_first_postorder(root): leftmost-deepest descendant.
- rb_next_postorder(node): if right-sibling exists: leftmost-deepest of right-sibling; else parent.
- rbtree_postorder_for_each_entry_safe: safe-against-removal-during-iter macro.

REQ-17: Caller-driven insertion pattern:
```
struct rb_node **link = &root.rb_node, *parent = NULL;
while (*link) {
    struct mytype *this = rb_entry(*link, struct mytype, node);
    parent = *link;
    if (cmp(new_key, this.key) < 0) link = &(*link).rb_left;
    else                            link = &(*link).rb_right;
}
rb_link_node(&new.node, parent, link);
rb_insert_color(&new.node, &root);
```

REQ-18: Augmented rbtree (rb_augment_callbacks):
- propagate(node, stop): per-node-edit re-propagate (e.g. subtree-max).
- copy(old, new): per-replace-due-to-rotation copy callback.
- rotate(old, new): per-rotation augment callback.
- Used by mm interval-trees, lockdep, sched/deadline rb_dl_tree.

## Acceptance Criteria

- [ ] AC-1: `rb_insert_color` after BST attachment preserves all 5 RB invariants.
- [ ] AC-2: `rb_erase` preserves all 5 RB invariants and removes node from tree.
- [ ] AC-3: `rb_first(empty)` = NULL; `rb_first(non-empty)` = leftmost node.
- [ ] AC-4: `rb_last(empty)` = NULL; `rb_last(non-empty)` = rightmost node.
- [ ] AC-5: In-order traversal via repeated `rb_next` starting at `rb_first` visits every node exactly once, in sorted key order.
- [ ] AC-6: `rb_prev` is exact inverse of `rb_next`.
- [ ] AC-7: `rb_replace_node(victim, new, root)`: in-order position preserved; readers via parent see new only after parent-link update.
- [ ] AC-8: `rb_replace_node_rcu`: concurrent reader sees either victim or new — never partial state.
- [ ] AC-9: Tree depth ≤ 2 · log2(N + 1) for all N ≤ 10^6 in stress test.
- [ ] AC-10: `rbtree_postorder_for_each_entry_safe` visits every node once with parent visited last (suitable for destroy + free).
- [ ] AC-11: `rb_root_cached`: rb_leftmost updated after every insert / erase to point at current leftmost.
- [ ] AC-12: `rb_root_linked`: prev/next list stable under insert / erase; in-order walk via list matches rb_next walk.
- [ ] AC-13: Augmented-rbtree callbacks invoked on every structural change; propagate-stop honored.
- [ ] AC-14: `__rb_erase_color` separately callable by augmented variants and produces same result as built-in.

## Architecture

```
struct RbNode {
    __rb_parent_color: usize,         // (parent_ptr | color) — low bit is color
    rb_right: *RbNode,
    rb_left: *RbNode,
}

struct RbRoot {
    rb_node: *RbNode,
}

struct RbRootCached {
    rb_root: RbRoot,
    rb_leftmost: *RbNode,
}

struct RbNodeLinked {
    node: RbNode,
    prev: *RbNodeLinked,
    next: *RbNodeLinked,
}

struct RbRootLinked {
    rb_root: RbRoot,
    rb_leftmost: *RbNodeLinked,
}

struct RbAugmentCallbacks {
    propagate: fn(node: *RbNode, stop: *RbNode),
    copy:      fn(old:  *RbNode, new:  *RbNode),
    rotate:    fn(old:  *RbNode, new:  *RbNode),
}

const RB_RED:   usize = 0;
const RB_BLACK: usize = 1;
```

`RbNode::link(node, parent, link)` (inline helper, caller-driven BST descent):
1. node.__rb_parent_color = parent as usize.        // RED (low-bit 0)
2. node.rb_left = null.
3. node.rb_right = null.
4. *link = node.                                     // patch into &parent.rb_left or rb_right

`RbTree::insert_color(node, root)`:
1. /* node already attached as RED leaf via RbNode::link */
2. __rb_insert(node, root, dummy_rotate):
3. loop:
   - parent = rb_parent(node).
   - if parent.is_null(): set node BLACK; return (became root).
   - if rb_color(parent) == BLACK: return.
   - gp = rb_parent(parent).
   - if parent == gp.rb_left:
     - u = gp.rb_right.
     - if u ∧ rb_color(u) == RED:
       - /* Case 1: uncle RED — recolor + move up */
       - rb_set_black(parent); rb_set_black(u); rb_set_red(gp).
       - node = gp; continue.
     - if node == parent.rb_right:
       - /* Case 2: inner — left-rotate parent → outer */
       - __rb_rotate_set_parents(parent, node, root, RB_RED) via rotate-left.
       - parent = node.
     - /* Case 3: outer — recolor + right-rotate gp */
     - rb_set_black(parent); rb_set_red(gp).
     - rotate-right(gp).
     - return.
   - else: mirror.

`RbTree::erase(node, root)`:
1. /* Standard BST erase (3 cases on #children) */
2. rebalance = __rb_erase(node, root, dummy_callbacks):
   - case A: no left child:
     - replace node with rb_right; rebalance = (node BLACK ? rb_right's parent : NULL).
   - case B: no right child:
     - replace node with rb_left; rebalance = (node BLACK ? rb_left's parent : NULL).
   - case C: two children:
     - successor = leftmost(rb_right).
     - splice successor into node's position (taking node's color); recursively handle successor's right child (case A relative to successor's parent).
3. if rebalance.is_some(): RbTree::erase_color(rebalance, root).

`RbTree::erase_color(parent, root)`:
1. loop:
   - node = (loop carries the "double-black" position).
   - if rb_color(node) == RED ∨ node is root: rb_set_black(node); return.
   - sibling = (node == parent.rb_left) ? parent.rb_right : parent.rb_left.
   - Case 1: sibling RED:
     - recolor sibling BLACK, parent RED; rotate parent toward node-side.
     - sibling = new sibling.
   - Case 2: sibling BLACK + both children BLACK:
     - rb_set_red(sibling); node = parent; parent = rb_parent(node); continue.
   - Case 3: sibling BLACK + outer-child BLACK + inner-child RED:
     - recolor inner BLACK, sibling RED; rotate sibling toward outer-side.
     - sibling = new sibling.
   - Case 4: sibling BLACK + outer-child RED:
     - sibling.color = parent.color; rb_set_black(parent); rb_set_black(outer).
     - rotate parent toward node-side; set node = root; return.

`RbTree::first(root) -> *RbNode` (inline):
1. n = root.rb_node.
2. if n.is_null(): return null.
3. while !n.rb_left.is_null(): n = n.rb_left.
4. return n.

`RbTree::next(node) -> *RbNode`:
1. if RB_EMPTY_NODE(node): return null.
2. if !node.rb_right.is_null():
   - n = node.rb_right.
   - while !n.rb_left.is_null(): n = n.rb_left.
   - return n.
3. while (parent = rb_parent(node)) ∧ node == parent.rb_right:
   - node = parent.
4. return parent.

`RbTree::replace_node(victim, new, root)`:
1. parent = rb_parent(victim).
2. /* Copy children + color */
3. new.__rb_parent_color = victim.__rb_parent_color.
4. new.rb_left = victim.rb_left.
5. new.rb_right = victim.rb_right.
6. /* Update children's parent */
7. if !victim.rb_left.is_null(): rb_set_parent(victim.rb_left, new).
8. if !victim.rb_right.is_null(): rb_set_parent(victim.rb_right, new).
9. /* Patch parent's link */
10. if !parent.is_null():
    - if parent.rb_left == victim: parent.rb_left = new.
    - else: parent.rb_right = new.
11. else: root.rb_node = new.

`RbTreeCached::insert_color(node, root_cached, leftmost: bool)` (inline):
1. if leftmost: root_cached.rb_leftmost = node.
2. RbTree::insert_color(node, &root_cached.rb_root).

`RbTreeCached::erase(node, root_cached)` (inline):
1. let next = if root_cached.rb_leftmost == node { Some(RbTree::next(node)) } else { None };
2. if let Some(n) = next: root_cached.rb_leftmost = n.
3. RbTree::erase(node, &root_cached.rb_root).

`RbTree::first_postorder(root) -> *RbNode`:
1. if root.rb_node.is_null(): return null.
2. return rb_left_deepest_node(root.rb_node).

`RbTree::next_postorder(node) -> *RbNode`:
1. if node.is_null(): return null.
2. parent = rb_parent(node).
3. if !parent.is_null() ∧ node == parent.rb_left ∧ !parent.rb_right.is_null():
   - return rb_left_deepest_node(parent.rb_right).
4. return parent.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rb_root_is_black` | INVARIANT | per-tree: root.rb_node.is_null() ∨ rb_color(root.rb_node) == BLACK. |
| `no_red_red_chain` | INVARIANT | per-node: rb_color(p) == RED ⟹ rb_color(p.parent) == BLACK. |
| `equal_black_height` | INVARIANT | per-tree: every root-to-NULL path has same black count. |
| `parent_links_consistent` | INVARIANT | per-node: rb_parent(p).rb_left == p ∨ rb_parent(p).rb_right == p. |
| `low_bit_is_color_only` | INVARIANT | per-node: __rb_parent_color & 1 ∈ {RB_RED, RB_BLACK}; high bits aligned ptr. |
| `rb_first_is_leftmost` | INVARIANT | per-tree: rb_first(root) == leftmost in-order node. |
| `rb_next_in_order` | INVARIANT | per-iter: rb_next yields nodes in sorted key order. |
| `cached_leftmost_consistent` | INVARIANT | per-RbRootCached: rb_leftmost == rb_first(&rb_root). |

### Layer 2: TLA+

`lib/rbtree.tla`:
- Per-insert / per-erase / per-rotate-left / per-rotate-right / per-recolor / per-iterate.
- Properties:
  - `safety_all_RB_invariants_preserved` — per-op: I1-I5 hold post-op.
  - `safety_depth_bounded_2log` — per-state: depth ≤ 2·log2(N+1).
  - `safety_in_order_traversal_sorted` — per-iter: rb_first + rb_next* yields sorted key sequence.
  - `safety_replace_node_preserves_in_order_position` — per-replace: rb_next / rb_prev sequence unchanged modulo identity.
  - `safety_rcu_replace_no_torn_observation` — per-replace_node_rcu: reader-via-parent observes victim ∨ new atomically.
  - `liveness_insert_erase_terminates` — per-op: fixup loop terminates (bounded by tree depth).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `RbTree::insert_color` post: all 5 RB invariants hold | `RbTree::insert_color` |
| `RbTree::erase` post: node not reachable from root; all 5 RB invariants hold | `RbTree::erase` |
| `RbTree::first` post: returns leftmost or NULL | `RbTree::first` |
| `RbTree::next` post: returns in-order successor or NULL | `RbTree::next` |
| `RbTree::prev` post: returns in-order predecessor or NULL | `RbTree::prev` |
| `RbTree::replace_node` post: in-order position of new == prior position of victim | `RbTree::replace_node` |
| `RbTreeCached::erase` post: rb_leftmost = new leftmost | `RbTreeCached::erase` |
| `RbTree::erase_color` post: I5 restored (black-height equalized) | `RbTree::erase_color` |
| `RbTree::next_postorder` post: parent visited after both children | `RbTree::next_postorder` |

### Layer 4: Verus/Creusot functional

`Per-rbtree insert / erase / lookup / iterate / augment` semantic equivalence: per-`Documentation/core-api/rbtree.rst` plus `lib/rbtree_test.c` randomized stress harness (insertion + deletion sequences validated against reference sorted-array model; depth-bound asserted).

## Hardening

(Inherits row-1 features from `lib/00-overview.md` § Hardening.)

rbtree reinforcement:

- **Per-RB-invariants strictly preserved on every op** — defense against per-degenerate-to-linked-list O(N) lookups.
- **Per-depth ≤ 2·log2(N+1)** — defense against per-stack-overflow on recursive walks (kernel uses iterative).
- **Per-low-bit-color encoding requires ≥ 4-byte alignment** — defense against per-pointer-mistagging (struct rb_node has __aligned(...) on some archs).
- **Per-`rb_replace_node_rcu` uses rcu_assign_pointer** — defense against per-reader-torn observation during replace.
- **Per-iterative fixup loops** — defense against per-deep-recursion stack blowup (insert/erase use loops, not recursion).
- **Per-augmented callbacks invoked on every structural change** — defense against per-stale-augment-data (interval-tree subtree-max correctness).
- **Per-cached leftmost updated on insert/erase** — defense against per-O(log N) priority-queue regression for sched / deadline / fair classes.
- **Per-postorder destroy macro** — defense against per-recursion-on-free (kernel paths use iterative postorder).
- **Per-`RB_EMPTY_NODE` sentinel after init / erase** — defense against per-double-erase / use-after-erase.
- **Per-rb_link_node initializes children to NULL** — defense against per-uninit-pointer dereference on first rebalance.
- **Per-`rb_parent` masks low bit** — defense against per-stale-color contamination of parent ptr.
- **Per-`__rb_erase_color` exposed for augmented variants** — defense against per-augment-reimplementation drift.
- **Per-tree contents are intrusive (no allocation)** — defense against per-OOM during structural op (alloc-free fastpath).

## Grsecurity/PaX-style Reinforcement

Baseline (apply to every Tier-3 surface):

- **PAX_USERCOPY** — rbtree itself does no user copies; embedding objects' usercopy discipline applies.
- **PAX_KERNEXEC** — no writable text; augmented-tree callbacks reside in `.text`.
- **PAX_RANDKSTACK** — caller's syscall entry randomizes stack base.
- **PAX_REFCOUNT** — rb_node has no internal refcount; embedding object's refcount saturates.
- **PAX_MEMORY_SANITIZE** — rbtree is intrusive; nothing allocated by rbtree code — sanitization is the embedder's responsibility.
- **PAX_UDEREF** — no user-pointer deref.
- **PAX_RAP / kCFI** — augmented-tree `rb_augment_callbacks` op-table type-tagged.
- **GRKERNSEC_HIDESYM** — `rb_*`, `__rb_erase_color`, `__rb_insert` hidden from non-CAP_SYSLOG kallsyms.
- **GRKERNSEC_DMESG** — `WARN_ON(RB_EMPTY_NODE)` gated behind CAP_SYSLOG.

Subsystem-specific reinforcement:

- **Intrusive design, no allocation** — `rb_link_node` / `rb_insert_color` cannot fail; embedding objects must already be allocated. PaX exploits this for the alloc-free fast path: structural ops in atomic/IRQ context cannot OOM, denying allocation-failure side channels.
- **Color-balancing invariants** — `__rb_insert` / `__rb_erase_color` maintain red-black invariants (every path same black depth, no two consecutive reds); `CONFIG_DEBUG_RBTREE` (PaX-promoted to BUG()) verifies on every structural mutation.
- **`__rb_erase_color` exposed for augmented variants** — re-implementations are forbidden; subsystems (interval tree, deadline IO, EDF) build on the canonical helpers, eliminating drift where a fork would silently miss a fix.
- **Parent-back-pointer integrity** — `rb_parent`/`__rb_parent_color` packs parent pointer + color into one word with low-bit tag; PaX asserts pointer alignment on every read, refusing torn writes from concurrent mutators (rbtree requires external synchronization).
- **Rationale** — rbtree backs sched-entities, EDF deadlines, interval trees, virtual memory mappings; corruption is silent and catastrophic. Mandatory DEBUG promotion + intrusive-only API + canonical helper-only mutation make corruption immediately fatal.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm VMA-tree (covered in `mm/vma-tree.md` if expanded; or maple-tree successor for 6.x)
- Scheduler EEVDF / CFS rb_root_cached uses (covered in `kernel/sched/fair.md` Tier-3)
- ext4 / btrfs extent rbtrees (covered in their respective fs Tier-3 docs)
- epoll rbroot (covered in `fs/eventpoll.md` if expanded)
- Augmented interval-tree (`include/linux/interval_tree_generic.h`) — generated header
- Implementation code

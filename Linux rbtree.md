### 维基百科定义
+ 节点是红色或黑色。
+ 根是黑色。
+ 所有叶子都是黑色（叶子是NIL节点）。
+ 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
+ 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。

### 作用
为了解决普通二叉树可能会退化成链条,使查找速度降低,在每次插入或删除操作树时做一些额外操作,使树维持平衡,提升查找速度

### 节点结构
```c
// 因为节点color非红即黑,只需要一位就能表示,0表示红,1表示黑;
// 这里把父节点指针和当前节点的color 放在了一个属性__rb_parent_color里,然后在取值时用位操作来分别取值;
// 可能对于系统关键数据结构,需要牺牲一定的代码可读性,换取空间使用率和效率的提升
struct rb_node {
	unsigned long  __rb_parent_color;
	struct rb_node *rb_right;
	struct rb_node *rb_left;
}
```

#### 插入操作
以eventpoll 使用rbtree的例子
```c
// fs/eventpoll.c
static void ep_rbtree_insert(struct eventpoll *ep, struct epitem *epi)
{
	int kcmp;
	struct rb_node **p = &ep->rbr.rb_root.rb_node, *parent = NULL;
	struct epitem *epic;
	bool leftmost = true;

    // 插入新的节点首先得找到插入位置,这与普通得二叉树搜寻没有区别,
    // 从根节点开始寻找,比当前节点小则往左子节点下钻,比当前节点大则往右子节点下钻,直到叶子节点(null)
	while (*p) {
		parent = *p;
		epic = rb_entry(parent, struct epitem, rbn);
		kcmp = ep_cmp_ffd(&epi->ffd, &epic->ffd);
		if (kcmp > 0) {
			p = &parent->rb_right;
			leftmost = false;
		} else
			p = &parent->rb_left;
	}

    // 接下来是tbtree提供得接口函数,rb_link_node,初始化插入的节点(红);
    // 即将插入节点的`父节点指针`指向父节点,然后初始化左右两个null子节点
	rb_link_node(&epi->rbn, parent, p);

    // 节点已插入,但不一定满足rbtree的特性,按需进行调整
	rb_insert_color_cached(&epi->rbn, &ep->rbr, leftmost);
}

// 一层封装
void rb_insert_color_cached(struct rb_node *node,
			    struct rb_root_cached *root, bool leftmost)
{
	__rb_insert(node, &root->rb_root, leftmost,
		    &root->rb_leftmost, dummy_rotate);
}
```

```c
static __always_inline void
__rb_insert(struct rb_node *node, struct rb_root *root,
	    bool newleft, struct rb_node **leftmost,
	    void (*augment_rotate)(struct rb_node *old, struct rb_node *new))
{
	struct rb_node *parent = rb_red_parent(node), *gparent, *tmp;

	if (newleft)
		*leftmost = node;

	while (true) {
		// 当前节点没有父节点的情况(即插入了是第一个节点,或者递归到了根几点)
		if (unlikely(!parent)) {
			// 这个函数名....
			// 其实这个函数是把当前节点(即node)的父节点指针设置为NULL,
			// 并把当前节点的颜色设置为RB_BLACK
			rb_set_parent_color(node, NULL, RB_BLACK);
			break;
		}

		// 如果父节点是黑色节点,则不用做什么操作也能满足红黑树条件
		if(rb_is_black(parent))
			break;

		// 逻辑运行到了这里说明parent节点是红节点了,这里rb_red_parent函数用来取出"红色节点"的父节点,即祖父节点gparent,这里根据规则也隐含了"祖父节点"是黑色的信息
		gparent = rb_red_parent(parent);

		tmp = gparent->rb_right;

		if (parent != tmp) {// 这里判断"父节点"是"祖父节点"的"左子节点"情况;
		// 此时tmp 是"祖父节点"的"右子节点",称为"叔父节点"

			// 叔父节点不为NULL 且 叔父节点是 红色节点时
			if (tmp && rb_is_red(tmp)) {
				/*
				 *
				 *      黑G            红G
				 *      / \            / \
				 *    红P  红U  -->  黑P  黑U
				 *    /             /
				 *  红n            红n
				 */

				// 设置父节点和叔父节点为黑
				rb_set_parent_color(tmp, gparent, RB_BLACK);
				rb_set_parent_color(parent, gparent, RB_BLACK);

				// 设置祖父节点为红,这时再当前祖父节点以下的树中,是已经满足了红黑树的特性,不需要再做处理;
				// 但当前祖父节点的父节点不一定黑,可能导致违反"节点不连续为红"的特性,需要把node指向gparent,把parent指向gparent的父节点,递归这次操作.

				node = gparent;
				parent = rb_parent(node);
				rb_set_parent_color(node, parent, RB_RED);
				continue;
			}

			// 逻辑到了这里,叔父节点肯定是黑的了(而且一定是NULL?)
			tmp = parent->rb_right;
			// 当前节点是父节点的右子节点时
			if (node == tmp) {
				/*
				 *
				 *      黑G            黑G
				 *      / \            / \
				 *    红P  黑U  -->   红n  黑U
				 *    / \             / \
				 * 	某1  红n        红P  某3
				 *		 / \       / \
				 *     某2  某3   某1 某2
				 *
				 */
				// 将 P 向左旋转得到新的树
				tmp = node->rb_left;

				// 注:这里WRITE_ONCE宏是为了使操作原子化
				WRITE_ONCE(parent->rb_right, tmp);
				WRITE_ONCE(node->rb_left, parent);
				if (tmp)
					rb_set_parent_color(tmp, parent,
							    RB_BLACK);
				rb_set_parent_color(parent, node, RB_RED);

				// 这是个dummy函数
				augment_rotate(parent, node);

			    // 将parent指针指向新的树的红N节点,
				// 这时可以看作当前node是父节点的左子节点,
				//
				// 此处应该把node指针指向原parent指针指向的结构,使逻辑完整,但后面没有再用到node,所以就省略了
				parent = node;
				tmp = node->rb_right;
			}

			// 下面是当前节点是父节点的左子节点的逻辑,对祖父节点向右旋转,并将原父节点设置为黑,原祖父节点设置为红;
			// 此时变化的子树的所有红黑树要满足的性质没有发生改变,故可以结束循环
			/*
			 *
			 *       黑G           黑P
			 *       / \          / \
			 *     红P  黑U  --> 红n  红g
			 *     /                  \
			 *    红n                  黑U
			 */
			WRITE_ONCE(gparent->rb_left, tmp); /* == parent->rb_right */
			WRITE_ONCE(parent->rb_right, gparent);
			if (tmp)
				rb_set_parent_color(tmp, gparent, RB_BLACK);
			__rb_rotate_set_parents(gparent, parent, root, RB_RED);
			augment_rotate(gparent, parent);
			break;
		} else { // 这里父节点是祖父节点的右子节点的情况

			// 叔父节点是祖父节点的左子节点,tmp指向叔父节点
			tmp = gparent->rb_left;
			if (tmp && rb_is_red(tmp)) {
				// 叔父节点存在 且 叔父节点是红色的时候;
				// 1.翻转父节点和叔父节点的颜色(红->黑);
				// 2.翻转祖父节点的颜色(黑->红);
				// 3.因为祖父的父节点不清楚是红是黑,需要把当前节点指针node指向祖父节点,parent指针指向祖父节点的父节点,递归整个逻辑;
				rb_set_parent_color(tmp, gparent, RB_BLACK);
				rb_set_parent_color(parent, gparent, RB_BLACK);
				node = gparent;
				parent = rb_parent(node);
				rb_set_parent_color(node, parent, RB_RED);
				continue;
			}

			// 叔父节点是黑节点时
			tmp = parent->rb_left;
			if (node == tmp) {
				// 当前节点是父节点的左子节点时,
				// 把父节点向右翻转转换成当前节点是父节点的右子节点,再接着执行相应逻辑

				tmp = node->rb_right;
				WRITE_ONCE(parent->rb_left, tmp);
				WRITE_ONCE(node->rb_right, parent);
				if (tmp)
					rb_set_parent_color(tmp, parent,
							    RB_BLACK);
				rb_set_parent_color(parent, node, RB_RED);
				augment_rotate(parent, node);
				parent = node;
				tmp = node->rb_left;
			}

			// 当前节点是父节点的右子节点时,祖父节点向左翻转,并翻转当前节点的父,叔父节点为红,祖父节点为黑
			WRITE_ONCE(gparent->rb_right, tmp); /* == parent->rb_left */
			WRITE_ONCE(parent->rb_left, gparent);
			if (tmp)
				rb_set_parent_color(tmp, gparent, RB_BLACK);
			__rb_rotate_set_parents(gparent, parent, root, RB_RED);
			augment_rotate(gparent, parent);
			break;
		}
	}
}
```
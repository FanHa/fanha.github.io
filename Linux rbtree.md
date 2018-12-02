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

#### 删除操作 
删除操作先通过普通的二叉树查找找到要删除的node节点的位置
```c
void rb_erase_cached(struct rb_node *node, struct rb_root_cached *root)
{
	struct rb_node *rebalance;
	// 删除节点
	rebalance = __rb_erase_augmented(node, &root->rb_root,
					 &root->rb_leftmost, &dummy_callbacks);

	// 删除节点后如果需要平衡,则再执行平衡操作
	if (rebalance)
		____rb_erase_color(rebalance, &root->rb_root, dummy_rotate);
}
```

```c
// 这里传进来的 "const struct rb_augment_callbacks *augment" 是个哑的 dummy_rotate,所以后面遇到augment相关调用直接省略
static __always_inline struct rb_node *
__rb_erase_augmented(struct rb_node *node, struct rb_root *root,
		     struct rb_node **leftmost,
		     const struct rb_augment_callbacks *augment)
{
	// child 指向要删除的节点node 的右子孩子
	struct rb_node *child = node->rb_right;
	// tmp 指向要删除的节点node 的左子孩子
	struct rb_node *tmp = node->rb_left;
	struct rb_node *parent, *rebalance;
	unsigned long pc;

	if (leftmost && node == *leftmost)
		*leftmost = rb_next(node);

	if (!tmp) {
		// 左子孩子节点为NULL节点
		pc = node->__rb_parent_color;
		parent = __rb_parent(pc);
		// 用node节点的右子节点替换到node的位置
		__rb_change_child(node, child, parent, root);
		if (child) {

			// 如果本来的右子节点不是NULL节点,则将右子节点的父节点指针 和 颜色变换为了要删除的节点的父节点指针和颜色;
			child->__rb_parent_color = pc;

			// 此时没有改变红黑树的特性,所以不需要平衡
			rebalance = NULL;
		} else

			// 当右子节点也为NULL节点时,如果要删除的节点时黑,则红黑树的特性失效,需要rebalance
			rebalance = __rb_is_black(pc) ? parent : NULL;
		tmp = parent;
	} else if (!child) {
		// 左孩子节点不为NULL, 右孩子节点为NULL,和前面左子NULL,右子不为NULL类似
		tmp->__rb_parent_color = pc = node->__rb_parent_color;
		parent = __rb_parent(pc);
		__rb_change_child(node, tmp, parent, root);
		rebalance = NULL;
		tmp = parent;
	} else {

		// 两个子节点都不为NULL时;
		// successor指向右子节点;
		struct rb_node *successor = child, *child2;

		// tmp指向右子节点的左孩子节点
		tmp = child->rb_left;

		// 这里是找node节点的右子树的最小值来继承被删除的node节点
		if (!tmp) {
			
			//node 节点右子节点successor的左子节点为NULL时,successor就是被删除的节点的继承节点
			/*
			 *
			 *    (n)          (s)
			 *    / \          / \
			 *  (x) (s)  ->  (x) (c)
			 *        \
			 *        (c)
			 */
			parent = successor;
			child2 = successor->rb_right;
		} else {
			// successor 节点的左子节点不为NULL时,则往左子树里找继承节点
			/*
			 *    (n)          (s)
			 *    / \          / \
			 *  (x) (y)  ->  (x) (y)
			 *      /            /
			 *    (p)          (p)
			 *    /            /
			 *  (s)          (c)
			 *    \
			 *    (c)
			 */
			do {
				parent = successor;
				successor = tmp;
				tmp = tmp->rb_left;
			} while (tmp);
			child2 = successor->rb_right;
			WRITE_ONCE(parent->rb_left, child2);
			WRITE_ONCE(successor->rb_right, child);
			rb_set_parent(child, successor);
		}

		// 用新的successor节点替代旧的node节点
		tmp = node->rb_left;
		WRITE_ONCE(successor->rb_left, tmp);
		rb_set_parent(tmp, successor);

		pc = node->__rb_parent_color;
		tmp = __rb_parent(pc);
		__rb_change_child(node, successor, tmp, root);

		// 因为继承节点肯定没有左子节点,所以如果继承节点的 右子节点不为NULL, 则一定是红色节点
		if (child2) {
			// 当继承节点的右子节点不为NULL时,继承节点一定是黑色节点,所以替换掉继承节点的右子节点要变成黑色;然后继承节点变成被删除的节点node的颜色
			successor->__rb_parent_color = pc;
			rb_set_parent_color(child2, parent, RB_BLACK);
			rebalance = NULL;
		} else {

			// 当继承节点的右子节点为NULL时,继承节点变为被删除节点的颜色
			unsigned long pc2 = successor->__rb_parent_color;
			successor->__rb_parent_color = pc;
			// 如果继承节点本身是黑色,则树失去了特性,需要被重新平衡,
			// 从继承节点本来的parent节点开始(实际这里是从这个parent以下,少了个黑色节点)
			rebalance = __rb_is_black(pc2) ? parent : NULL;
		}
		tmp = successor;
	}
	return rebalance;
}
```
#### 再平衡
这里传进来的augment_rotate参数同样是个哑函数,后面省略;
在执行这个函数前的背景是:
+ parent的左子树的路径少了一个黑色节点;
+ parent本身是黑色节点;(如果是红色,早再上一步就已经翻转成黑色,不需要接着执行到这里);

```c
static __always_inline void
____rb_erase_color(struct rb_node *parent, struct rb_root *root,
	void (*augment_rotate)(struct rb_node *old, struct rb_node *new))
{
	struct rb_node *node = NULL, *sibling, *tmp1, *tmp2;

	while (true) {
		// 用递归向上冒泡
		// 取sibling指针为parent节点的右子节点(node 为 parent的左子节点)
		sibling = parent->rb_right;
		if (node != sibling) {	/* node == parent->rb_left */
			if (rb_is_red(sibling)) {
				// sibling节点为红色时,把parent节点向左翻转,这时整个在下图中的子树的红P节点到左子树的叶子节点的黑色数并没有变化,依然比预定目标少1;
				//我们把sibling节点指向本来的Sl,这样就形成了sibling为黑色节点的条件,
				/*
				 *
				 *      黑P               黑S
				 *     /  \              /  \
				 *   黑N  红S    -->    红P  黑Sr
				 *        /  \         /  \
				 *      黑Sl 黑Sr     黑N  黑Sl --->新sibling指向这个位置
				 */
				tmp1 = sibling->rb_left;
				WRITE_ONCE(parent->rb_right, tmp1);
				WRITE_ONCE(sibling->rb_left, parent);
				rb_set_parent_color(tmp1, parent, RB_BLACK);
				__rb_rotate_set_parents(parent, sibling, root,
							RB_RED);
				augment_rotate(parent, sibling);
				sibling = tmp1;
			}

			// sibling为黑色节点时的执行逻辑如下
			tmp1 = sibling->rb_right;
			if (!tmp1 || rb_is_black(tmp1)) {

				//sibling 的右子节点是黑色时;
				tmp2 = sibling->rb_left;
				if (!tmp2 || rb_is_black(tmp2)) {
					// sibling 的左右子节点都是黑色时
					/*
					 *
					 *    (p)             (p)
					 *    / \             / \
					 *  黑N  黑S    --> 黑N  红s
					 *      / \             /  \
					 *    黑Sl 黑Sr       黑Sl  黑Sr
					 */
					 // 将sibling节点由红变黑
					rb_set_parent_color(sibling, parent,
							    RB_RED);

					if (rb_is_red(parent))
						// 如果parent节点本来是红的,则转黑;
						// 此时p节点往左子树方向的路径+1,右子树方向不变,平衡了前面左子树路径少1的缺憾,树已平衡,可以直接break;
						rb_set_black(parent);
					else {
						// 如果parent本来是黑的;
						// 此时parent的左子树路径黑色依然少1,右子树路径经过黑转红后也少1,整个parent子树的路径黑色都少1;
						// 把node指向parent节点,parent指向parent的父节点,
						// 可以看成一个新的子树路径黑色少1 的情况,continue递归向上冒泡再走一遍逻辑;
						node = parent;
						parent = rb_parent(node);
						if (parent)
							continue;
					}
					break;
				}
				// sibling右子节点为黑色,左子节点为红色时;
				// 此时依然是parent的左子树N节点的路径黑色少1;
				/*
				 * Case 3 - right rotate at sibling
				 * (p could be either color here)
				 *
				 *    (p)             (p)
				 *    / \             / \
				 * 黑N   黑S    -->  黑N  红sl     
				 *      /  \               \
				 *    红sl  黑Sr            黑S
				 *                           \
				 *                            黑Sr
				 *
				 */
				 //这里情况有点绕,因为经过这次转变后:
				 // 黑N子树路径黑色少1;
				 // 红sl左子树路径黑色少1;
				 // 但是经过接下来执行的逻辑一样是可以使这些路径黑色少1的子树恢复平衡.....
				tmp1 = tmp2->rb_right;
				WRITE_ONCE(sibling->rb_left, tmp1);
				WRITE_ONCE(tmp2->rb_right, sibling);
				WRITE_ONCE(parent->rb_right, tmp2);
				if (tmp1)
					rb_set_parent_color(tmp1, sibling,RB_BLACK);

				tmp1 = sibling;
				sibling = tmp2;
			}
			// 这里右两种可能:
			// 1.
			// sibling 的 右子节点为红色时,把p向左翻转(和颜色);
			// 此时子树左边的路径黑色+1;
			// 右边先-1(黑S),然后又+1(sr由红转黑);
			// 左右维持平衡,可以break;
			/*
			 *
			 *        (p)               (s)
			 *        / \               / \
			 *     黑N   黑S     -->  黑P  黑Sr
			 *           / \         / \
			 *        (sl) 红sr    黑N (sl)
			 */

			 // 2.
			 // 经过前面的"sibling右子节点为黑色,左子节点为红色时"操作得到的结果如下并且:
			 // 2.1 黑N子树路径黑色少1;
			 // 2.2 红sl左子树路径黑色少1.
			 // 对p向左翻转并变色黑,这些路径黑色少1的子树恢复了正常,也可以break出循环
			 /*
			 *    (p)             (sl)
			 *    / \             /  \
			 *  黑N  红sl   -->  黑P  黑S
			 *        \         /      \
			 *        黑S      黑N      黑Sr
			 *         \
			 *          黑Sr
			 */
			tmp2 = sibling->rb_left;
			WRITE_ONCE(parent->rb_right, tmp2);
			WRITE_ONCE(sibling->rb_left, parent);
			rb_set_parent_color(tmp1, sibling, RB_BLACK);
			if (tmp2)
				rb_set_parent(tmp2, parent);
			__rb_rotate_set_parents(parent, sibling, root,
						RB_BLACK);
			augment_rotate(parent, sibling);
			break;
		} else {
			// 这个与上面的逻辑基本镜像,即node为parent的右子节点,sibling为parent的左子节点;
			// 因为上面的循环可能会递归冒泡到上一层,而上一层的node 和 sibling的左右顺序并不一定是第一次进来时的左右顺序;
			sibling = parent->rb_left;
			if (rb_is_red(sibling)) {
				/* Case 1 - right rotate at parent */
				tmp1 = sibling->rb_right;
				WRITE_ONCE(parent->rb_left, tmp1);
				WRITE_ONCE(sibling->rb_right, parent);
				rb_set_parent_color(tmp1, parent, RB_BLACK);
				__rb_rotate_set_parents(parent, sibling, root,
							RB_RED);
				augment_rotate(parent, sibling);
				sibling = tmp1;
			}
			tmp1 = sibling->rb_left;
			if (!tmp1 || rb_is_black(tmp1)) {
				tmp2 = sibling->rb_right;
				if (!tmp2 || rb_is_black(tmp2)) {
					/* Case 2 - sibling color flip */
					rb_set_parent_color(sibling, parent,
							    RB_RED);
					if (rb_is_red(parent))
						rb_set_black(parent);
					else {
						node = parent;
						parent = rb_parent(node);
						if (parent)
							continue;
					}
					break;
				}
				/* Case 3 - left rotate at sibling */
				tmp1 = tmp2->rb_left;
				WRITE_ONCE(sibling->rb_right, tmp1);
				WRITE_ONCE(tmp2->rb_left, sibling);
				WRITE_ONCE(parent->rb_left, tmp2);
				if (tmp1)
					rb_set_parent_color(tmp1, sibling,
							    RB_BLACK);
				augment_rotate(sibling, tmp2);
				tmp1 = sibling;
				sibling = tmp2;
			}
			/* Case 4 - right rotate at parent + color flips */
			tmp2 = sibling->rb_right;
			WRITE_ONCE(parent->rb_left, tmp2);
			WRITE_ONCE(sibling->rb_right, parent);
			rb_set_parent_color(tmp1, sibling, RB_BLACK);
			if (tmp2)
				rb_set_parent(tmp2, parent);
			__rb_rotate_set_parents(parent, sibling, root,
						RB_BLACK);
			augment_rotate(parent, sibling);
			break;
		}
	}
}
```
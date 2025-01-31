# 使用 Unsafe

Rust 通常只给了我们以作用域的方式来限制不安全代码块的工具。不幸的是，现实要比这复杂得多。例如，考虑下面这个玩具函数：

```rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx < arr.len() {
        unsafe {
            Some(*arr.get_unchecked(idx))
        }
    } else {
        None
    }
}
```

这个函数是安全和正确的。我们先检查索引是否在界内，如果是，就以不检查的方式索引到数组中。我们说，这样一个正确实现的不安全函数是*健全*的，这意味着安全代码不能通过它引起未定义行为（记住，这是安全 Rust 的唯一基本属性）。

但即使在这样一个微不足道的函数中，不安全的代码块也是值得怀疑的，比如将`<`改为`<=`：

```rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx <= arr.len() {
        unsafe {
            Some(*arr.get_unchecked(idx))
        }
    } else {
        None
    }
}
```

这个程序现在是*不健全*的，安全的 Rust 会导致未定义行为，尽管*我们只修改了安全代码*。这就是安全的基本问题：它是非局部的。我们的不安全操作的健壮性必然取决于由其他“安全”操作建立的状态。

安全是模块化的，你不需要考虑任何其它的不安全块带来的潜在问题。例如，对一个分片做一个未经检查的索引并不意味着你突然需要担心这个分片是空的或者包含未初始化的内存。没有任何根本性的变化。然而，安全又*不是*模块化的，因为程序本身是有状态的，你的不安全操作可能依赖于任意的其他状态。

当我们加入实际的持久化状态时，这种非局部性会变得更糟糕。例如，让我们看一下`Vec`的一个简单实现：

```rust
use std::ptr;

// Note: This definition is naive. See the chapter on implementing Vec.
pub struct Vec<T> {
    ptr: *mut T,
    len: usize,
    cap: usize,
}

// Note this implementation does not correctly handle zero-sized types.
// See the chapter on implementing Vec.
impl<T> Vec<T> {
    pub fn push(&mut self, elem: T) {
        if self.len == self.cap {
            // not important for this example
            self.reallocate();
        }
        unsafe {
            ptr::write(self.ptr.add(self.len), elem);
            self.len += 1;
        }
    }
    # fn reallocate(&mut self) { }
}

# fn main() {}
```

这段代码很简单，可以很简单地确认和验证，但是现在我们添加以下方法：

<!-- ignore: simplified code -->
```rust,ignore
fn make_room(&mut self) {
    // grow the capacity
    self.cap += 1;
}
```

这段代码是 100% 安全的 Rust，但它也是完全不健全的。改变容量违反了 Vec 的不变性（即`cap`反映了 Vec 中分配的空间）。这不是 Vec 的其他部分所能防范的。它*不得不*相信容量字段，因为没有办法验证它。

因为它依赖于一个结构字段的不变性，这段“不安全”的代码不仅仅污染了整个函数：它污染了整个*模块*。一般来说，限制不安全代码的范围的唯一方法是在模块边界上设置权限。

然而，其实这个改动是可以*完美地*工作的。`make_room`的存在对于 Vec 的健全性来说*不是*个问题，因为我们没有把它标记为公共的。只有定义了这个函数的模块可以调用它。另外，`make_room`直接访问了 Vec 的私有字段，所以它只能写在与 Vec 相同的模块中。

因此，我们有可能基于复杂的不变性，编写一个完全安全的抽象。这对安全 Rust 和不安全 Rust 之间的关系是*非常重要*的。

我们已经看到，不安全代码必须*一部分*信任安全代码，但不应该完全信任安全代码。出于类似的原因，访问控制对不安全代码也很重要：它可以防止我们不得不信任宇宙中所有的安全代码，防止它们扰乱我们的信任状态。

安全万岁！

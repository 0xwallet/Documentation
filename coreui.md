# CoreUI

coreui 是一个基于 Bootstrap 的 UI 库, 完全兼容 Bootstrap 的语法.

## Grid 系统

Bootstrap 的移动端优先的布局设计, 是基于一个 十二列的布局系统, 5 种默认的响应行为, Sass 变量以及 mixins, 还有大量的预定义的类.

Bootstrap 的grid 系统使用了一系列的 containers, rows 和 columns 来对内容进行布局和对其. 它基于 flexbox 并且是完全响应式的. 下面是一个例子, 以及对 grid 的详细解析.

```html
<div class="container">
  <div class="row">
    <div class="col-sm">
      One of three columns
    </div>
    <div class="col-sm">
      One of three columns
    </div>
    <div class="col-sm">
      One of three columns
    </div>
  </div>
</div>
```

上面的例子创造了在任意大小的设备上等宽的三列. 这三列处于父元素`.container` 中, 并且在页面上居中.





## Modal

Modal 是 Boostrap 中的一种控件, 也就是俗称的弹窗.  Modal 使用 HTML, CSS 和 JavaScript 制作而成. 它们处于页面中所有其它的元素之前, 并且取代了 <body> 的滚动条, 这样 modal 的内容就可以滚动.

点击 modal 上的 "backdrop" 按钮, 就会自动关闭 modal.

Bootstrap 只支持一次显示一个 modal 窗口. 不支持嵌套 modals, 是因为我们相信这样会造成很差的用户体验.

Modal 使用 `position: fixed`, 有时这会影响到它的渲染. 尽可能地让你的 modal HTML 处于顶层的位置, 以避免对其它元素潜在的影响. 将 modal 嵌套在另一个 fixed 的元素内可能会有问题.

另外, 由于 `position: fixed`, modal 在移动端的使用可能会有问题.

由于 HTML5 的机制, `autofocus` 属性在 modal 内无效. 为了达到相同效果, 可以使用以下 JavaScript:

```js
$('#myModal').on('shown.bs.modal', function () {
  $('#myInput').trigger('focus')
})
```

### 例子

Modal 组件

下面是一个静态的 modal 的例子(它的 position 和 dispaly 都被覆盖了). 它包含了 modal header, modal body(处于 padding 的需要), 以及 modal footer(可选的). 尽可能在 modal header 上包含关闭按钮, 或者是提供另外的明显的关闭动作.

```html
<div class="modal" tabindex="-1" role="dialog">
  <div class="modal-dialog" role="document">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title">Modal title</h5>
        <button type="button" class="close" data-dismiss="modal" aria-label="Close">
          <span aria-hidden="true">&times;</span>
        </button>
      </div>
      <div class="modal-body">
        <p>Modal body text goes here.</p>
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-primary">Save changes</button>
        <button type="button" class="btn btn-secondary" data-dismiss="modal">Close</button>
      </div>
    </div>
  </div>
</div>
```

下面这个例子包含了用于触发 modal 的按钮:

```html
<!-- Button trigger modal -->
<button type="button" class="btn btn-primary" data-toggle="modal" data-target="#exampleModal">
  Launch demo modal
</button>

<!-- Modal -->
<div class="modal fade" id="exampleModal" tabindex="-1" role="dialog" aria-labelledby="exampleModalLabel" aria-hidden="true">
  <div class="modal-dialog" role="document">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title" id="exampleModalLabel">Modal title</h5>
        <button type="button" class="close" data-dismiss="modal" aria-label="Close">
          <span aria-hidden="true">&times;</span>
        </button>
      </div>
      <div class="modal-body">
        ...
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-dismiss="modal">Close</button>
        <button type="button" class="btn btn-primary">Save changes</button>
      </div>
    </div>
  </div>
</div>
```

当 modal 变得太长的时候, 用户可以对其进行滚动.

添加`.modal-dialog-centered` 到 `.modal-dialog` 里, 可以使 modal 垂直居中:

```html
<!-- Button trigger modal -->
<button type="button" class="btn btn-primary" data-toggle="modal" data-target="#exampleModalCenter">
  Launch demo modal
</button>

<!-- Modal -->
<div class="modal fade" id="exampleModalCenter" tabindex="-1" role="dialog" aria-labelledby="exampleModalCenterTitle" aria-hidden="true">
  <div class="modal-dialog modal-dialog-centered" role="document">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title" id="exampleModalLongTitle">Modal title</h5>
        <button type="button" class="close" data-dismiss="modal" aria-label="Close">
          <span aria-hidden="true">&times;</span>
        </button>
      </div>
      <div class="modal-body">
        ...
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-dismiss="modal">Close</button>
        <button type="button" class="btn btn-primary">Save changes</button>
      </div>
    </div>
  </div>
</div>
```

Modal 里面还可添加 tooltips(标识符) 和 popovers(提示框). 当 modal 关闭时, 所有的 tooltips 和 popovers 都会自动关闭.

```html
<div class="modal-body">
  <h5>Popover in a modal</h5>
  <p>This <a href="#" role="button" class="btn btn-secondary popover-test" title="Popover title" data-content="Popover body content is set in this attribute.">button</a> triggers a popover on click.</p>
  <hr>
  <h5>Tooltips in a modal</h5>
  <p><a href="#" class="tooltip-test" title="Tooltip">This link</a> and <a href="#" class="tooltip-test" title="Tooltip">that link</a> have tooltips on hover.</p>
</div>
```

只需要在 `.modal-body` 里嵌套 `.container-fluid` 元素, 就可以像往常一样使用 grid 系统了:

```html
<div class="modal-body">
  <div class="container-fluid">
    <div class="row">
      <div class="col-md-4">.col-md-4</div>
      <div class="col-md-4 ml-auto">.col-md-4 .ml-auto</div>
    </div>
    <div class="row">
      <div class="col-md-3 ml-auto">.col-md-3 .ml-auto</div>
      <div class="col-md-2 ml-auto">.col-md-2 .ml-auto</div>
    </div>
    <div class="row">
      <div class="col-md-6 ml-auto">.col-md-6 .ml-auto</div>
    </div>
    <div class="row">
      <div class="col-sm-9">
        Level 1: .col-sm-9
        <div class="row">
          <div class="col-8 col-sm-6">
            Level 2: .col-8 .col-sm-6
          </div>
          <div class="col-4 col-sm-6">
            Level 2: .col-4 .col-sm-6
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```


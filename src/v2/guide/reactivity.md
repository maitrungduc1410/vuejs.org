---
title: Reactivity in Depth
type: guide
order: 601
---

Bây giờ chúng ta sẽ cùng tìm hiểu sâu hơn về Vue! Một trong những tính năng khác biệt nhất của Vue là hệ thống phản ứng "không phô trương". Các models chỉ là các đối tượng Javascript thuần. Khi bạn chỉnh sửa chúng thì view sẽ thay đổi. Nó làm cho việc quản lý trạng thái đối tượng được đơn giản và trực quan hơn, nhưng điều quan trọng là vẫn nên hiểu cách hoạt động của nó để tránh một số sai lầm phổ biến. Trong phần này, chúng ta sẽ đi sâu vào một số chi tiết ở mức thấp hơn của hệ thống phản ứng của Vue.

## Các thay đổi trong Vue được theo dõi như thế nào

Khi bạn truyền một đối tượng thuần JavaScript cho một thực thể Vue như đối tượng `data` của nó, Vue sẽ duyệt qua tất cả các thuộc tính của nó và chuyển đổi chúng thành các getter / setters sử dụng [Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty). Đây là một tính năng chỉ có ở ES5 và khó để thay thế, đó là lý do tại sao Vue không hỗ trợ IE8 và các phiên bản thấp hơn của nó.

Các getter và setter có thể được nhìn thấy bởi người dùng, nhưng sâu hơn nữa, nó cho phép Vue thực hiện theo dõi các phụ thuộc và thay đổi thông báo khi các thuộc tính được truy cập hoặc sửa đổi. Một trong những hạn chế của việc này là trình duyệt set in ra định dạng getter / setters khác nhau khi chuyển đổi các đối tượng, do đó khuyến khích các bạn cài đặt [vue-devtools](https://github.com/vuejs/vue-devtools) để có một cái nhìn sâu hơn về ứng dụng của mình.

Every component instance has a corresponding **watcher** instance, which records any properties "touched" during the component's render as dependencies. Later on when a dependency's setter is triggered, it notifies the watcher, which in turn causes the component to re-render.

![Reactivity Cycle](/images/data.png)

## Hạn chế của việc phát hiện thay đổi

Do những hạn chế của JavaScript hiện tại (và việc `Object.observe` không còn được sử dụng tới), Vue **không thể phát hiện sự thêm hoặc xoá của một thuộc tính nào đó**. Vì Vue thực hiện quá trình chuyển đổi getter / setter trong quá trình khởi tạo thực thể, một thuộc tính bất kì phải được khai báo trong đối tượng `data` để Vue có thể thực hiện chuyển đổi và làm nó trở nên `reactive`. Ví dụ:

``` js
var vm = new Vue({
  data: {
    a: 1
  }
})
// `vm.a` giờ đã trở thành reactive

vm.b = 2
// `vm.b` không phải `reactive`
```

Vue không cho phép thêm động các thuộc tính `reactive` ở mức `root` vào một thực thể đã được tạo từ trước. Tuy nhiên, có thể thêm các thuộc tính `reactive` vào một đối tượng lồng nhau bằng cách sử dụng `Vue.set(object, key, value)`:

``` js
Vue.set(vm.someObject, 'b', 2)
```

Bạn cũng có thể sử dụng `vm.$set`, đây là một bí danh (alias) toàn cầu của `Vue.set`:

``` js
this.$set(this.someObject, 'b', 2)
```

Đôi khi bạn có thể muốn gán một số thuộc tính cho một đối tượng hiện có, ví dụ bằng cách sử dụng `Object.assign()` hoặc `_.extend()`. Tuy nhiên, các thuộc tính mới được thêm vào này sẽ không kích hoạt thay đổi (tức là khi chúng thay đổi Vue sẽ không re-render). Trong những trường hợp như thế, bạn nên tạo ra một đối tượng mới với các thuộc tính từ cả đối tượng gốc lẫn đối tượng có chứa các thuộc tính bạn muốn thêm vào, ví dụ:

``` js
// Không nên `Object.assign(this.someObject, { a: 1, b: 2 })`
this.someObject = Object.assign({}, this.someObject, { a: 1, b: 2 })
```

Đồng thời cũng vẫn còn tồn tại một số hạn chế liên quan đến Array, những điều đó chúng ta sẽ cùng thảo luộn ở [Render danh sách](list.html#Caveats).

## Khai báo thuộc tính Reactivity

Vì Vue không cho phép thêm động các thuộc tính phản ứng mức root, nên bạn phải khởi tạo các thực thể Vue bằng cách khai báo trước tất cả các thuộc tính dữ liệu reactive ở root, thậm chí với một giá trị rỗng:

``` js
var vm = new Vue({
  data: {
    // khai báo message với giá trị rỗng
    message: ''
  },
  template: '<div>{{ message }}</div>'
})
// thiết lập sau giá trị cho `message`
vm.message = 'Hello!'
```

Nếu bạn không khai báo `message` ở trong `data`, Vue sẽ cảnh báo bạn rằng `render function` đang cố gắng truy cập một thuộc tính không tồn tại.

Có những lý do kỹ thuật đằng sau hạn chế này - nó loại bỏ một lớp các trường hợp ngoại lệ trong hệ thống theo dõi phụ thuộc, và cũng làm cho các cá thể Vue hoạt động một cách tốt hơn với các hệ thống kiểm tra kiểu. Nhưng cũng có một cân nhắc quan trọng về khả năng bảo trì mã nguồn (code): đối tượng `data` giống như lược đồ cho trạng thái của các components. Khai báo trước tất cả các thuộc tính `reactive` làm cho code của bạn dễ hiểu hơn khi được xem lại sau hoặc được đọc bởi một nhà phát triển khác.

## Hàng đợi cập nhật không đồng bộ

Trong trường hợp bạn chưa để ý, Vue thực hiện update DOM **bất đồng bộ**. Bất cứ khi nào thay đổi dữ liệu được quan sát thấy, nó sẽ mở một hàng đợi và đưa tất cả các thay đổi dữ liệu xảy ra trong cùng một vòng lặp sự kiện. Nếu cùng một `watcher` được kích hoạt nhiều lần, nó sẽ được đưa vào hàng đợi duy nhất một lần. Việc loại bỏ trùng lặp đệm này rất quan trọng trong việc tránh các phép tính không cần thiết và các thao tác DOM thừa thãi. Sau đó, trong vòng lặp sự kiện tiếp theo được "đánh dấu" (tick) và Vue sẽ xóa hàng đợi cũ và thực hiện công việc thực tế (đã loại đi các tác vụ trùng lặp). Để bạn hiểu rõ hơn thì Vue cố gắng thực hiện `Promise.then` và `MessageChannel` cho hàng đợi bất đồng bộ và trả về callback `setTimeout(fn, 0)`.

Ví dụ, khi bạn thiết lập `vm.someData = 'new value'`, component sẽ không re-render ngay lập tức. Nó sẽ được update ở lần `tick` tiếp theo, khi hàng đợi bị xóa. Hầu hết thời gian chúng ta không cần phải quan tâm về điều này, nhưng nó có thể giúp ích khi bạn muốn làm điều gì đó phụ thuộc vào trạng thái của DOM sau khi cập nhật. Mặc dù Vue.js thường khuyến khích các nhà phát triển suy nghĩ theo cách "theo hướng dữ liệu" và tránh thao tác trực tiếp vào DOM, nhưng đôi khi chúng ta cũng nên thử điều gì đó mới mẻ. Để chờ cho đến khi Vue.js cập nhật xong DOM sau khi thay đổi dữ liệu, có thể sử dụng `Vue.nextTick(callback)` ngay lập tức sau khi thay đổi dữ liệu. Hàm callback sẽ được gọi sau khi DOM đã hoàn tất việc cập nhật. Ví dụ:

``` html
<div id="example">{{ message }}</div>
```

``` js
var vm = new Vue({
  el: '#example',
  data: {
    message: '123'
  }
})
vm.message = 'new message' // thay đổi dữ liệu
vm.$el.textContent === 'new message' // false
Vue.nextTick(function () {
  vm.$el.textContent === 'new message' // true
})
```

Ta cũng có phương thức `vm.$nextTick()`, có thể thao tác với các thành phần bên trong, bởi vì nó không cần đến đối tượng global `Vue` và callback của nó là `this` trong trường hợp này sẽ tự động được liên kết với đối tượng Vue hiện tại:

``` js
Vue.component('example', {
  template: '<span>{{ message }}</span>',
  data: function () {
    return {
      message: 'not updated'
    }
  },
  methods: {
    updateMessage: function () {
      this.message = 'updated'
      console.log(this.$el.textContent) // => 'không được cập nhật'
      this.$nextTick(function () {
        console.log(this.$el.textContent) // => 'được cập nhật'
      })
    }
  }
})
```

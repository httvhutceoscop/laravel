Model factories được ra mắt từ Laravel 5.1 với mục đích tạo ra các mô hình "giả" một cách nhanh chóng.

Điều này sẽ hữu ích trong 2 trường hợp là - test và tạo ra các database seeding. Trong bài viết này sẽ hướng dẫn các bạn hiểu sâu hơn về model factories thông qua 1 ví dụ cụ thể.

### Starting out

Giả sử bạn được thuê bởi dịch vụ sửa nhà LEMON, và họ muốn các khách hàng của họ gửi cho họ các vấn đề cho LEMON sau khi họ sửa xong nhà. Với cái tên như LEMON thì sẽ không tránh khỏi việc nhận các phàn nàn từ khách hàng. Và nhiệm vụ của bạn sẽ xây dựng quy trình này.

Và để xử lý, chúng ta sẽ sử dụng bảng `users` lưu thông tin khách hàng và tạo mới bảng `issues`.

Đầu tiên tạo 1 project mới:

```
laravel new lemon-support
```

Tiếp theo sẽ tạo model issues và migration từ lệnh `artisan`. Sử dụng lệnh sau với cờ `-m` hoặc `migration`:

```
php artisan make:model Issues -m
```

File `Issues.php` sẽ được tạo ra ở thư mục `app/`. Và bên trong thư mục `database/migration/` sẽ có file `create_issues_table`.

Tiếp theo chúng ta sẽ tùy chỉnh 1 chút file `app/User.php`. Chúng ta sẽ định nghĩa quan hệ issues:

```php
public function issues()
{
  return $this->hasMany('issues');
}
```

Lưu ý: nếu bạn không sử dụng homestead mặc định thì các bạn có thể sửa file `.env` để đổi sang database mà bạn muốn dùng.

Tiếp theo, chúng ta sẽ code cho các file migration.

### Tạo migrations
Sửa file `database/migrations/datetime_create_issues_tables.php` như đoạn code bên dưới:

```php
public function up()
{
  Schema::create('issues', function(Blueprint $table) {
    $table->increments('id');
    $table->integer('user_id'); // id của user
    $table->string('subject'); // tiêu đề của các issues
    $table->text('description'); // mô tả chi tiết issues
    $table->timestamp();
  });
}
```

Trong migration này, chúng ta đã tạo quan hệ 2 bảng `issues` và `users` thông qua cột `user_id`.

Chạy migration:

```
php artisan migrate
```

Bạn sẽ thấy các output như sau:

```
Migration table created successfully
```

Tiếp theo chúng ta sẽ sử dụng database seeding để tạo dữ liệu giả.

### Xây dựng Database Seeds
Database seeds được dùng để tạo ra các dữ liệu dummpy, sử dụng khi phát triển project hoặc mục đích testing. Điều này thật sự tiện lợi khi làm việc với giao diện người dùng hoặc với 1 nhóm.

Việc tạo ra database seeds cũng rất đơn giản với lệnh:

```
php artisan make:seeder UserTableSeeder
```

Tạo thêm 1 cái nữa cho bảng `issues`:

```
php artisan make:seeder IssueTableSeeder
```

Chúng ta sẽ phải sửa lại code của file `database/seeds/DatabaseSeeder.php` 1 chút:

```php
public function run()
{
  Model::unguard();

  $this->call(UserTableSeeder::class);
  $this->call(IssueTableSeeder::class);

  Model::reguard();
}
```

Tiếp theo chúng ta sẽ sử dụng model factories để hiểu hơn về cách mà các seeder thêm mới dữ liệu:

### Tạo các Model Factories
Trong Laravel, Eloquent hay các query builder cũng có thể tạo ra các seed data, tuy nhiên trong bài viết này sẽ nói về cách dùng model factories để tạo ra 1 model "dummy" mà có thể dùng cho cả việc seed data và testing.

Mở file `database/factories/ModelFactory.php` và nó có nội dung mặc định như bên dưới:

```php
$factory->define(App\User::class, function (Faker\Generator $faker) {
  return [
    'name' => $faker->name,
    'email' => $faker->email,
    'password' => bcrypt(str_random(10)),
    'remember_token' => str_random(10),
  ];
});
```

Giải thích 1 chút về đoạn code trên. Trong phương thức `define` của factory sẽ có 2 tham số. Tham số đầu tiên là model `App\User::class` và tham số thứ 2 là 1 callback với tham số là thư viện `Faker`. Mục đích của callback là tạo ra các dữ liệu giả, và thư viện `Faker` có thể được dùng với các kiểu dữ liệu khác nhau như `number` hay `string`. Ví dụ:

- $faker->name - "ToiLamIT"
- $faker->email - "toilamit.com@gmail.com"

**Chú ý**: nếu project đang trong quá trình sử dụng thì không nên dùng `$faker->email` mà hãy dùng `$faker->safeEmail` vì safeEmail chỉ sử udjng dạng `example.org` do [RFC2606](https://tools.ietf.org/html/rfc2606) cho mục đích kiểm thử.

Bây giờ chúng ta sẽ tạo 1 factory cho model `Issues`:

```php
$factory->define(App\Issues::class, function (Faker\Generator $faker) {
  return [
    'subject' => $faker->sentence(5),
    'description' => $faker->text(),
  ];
});
```

Ở đoạn code trên, khi chạy sẽ tạo ra 1 subject với 5 từ và description với 1 đoạn text giả.

Nào, bây giờ quay trở lại với các seed classes và dùng factories để tạo dữ liệu:

Sửa file `UserTableSeeder` như sau:

```php
public function run()
{
  factory(App\User::class, 2)->create()->each(function($u) {
    $u->issues()->save(factory(App\Issues::class)->make());
  });
}
```

Giải thích đoạn code trên. Đầu tiên sẽ tạo ra 2 users và lưu vào trong database, sau đó lặp qua mỗi 1 user và tiếp tục tạo ra các issues tương ứng cho từng user đã tạo.

Cuối cùng chúng ta chạy lệnh sau:

```
php artisan migrate --seed
```

Output sẽ là:

```
Migrated: 2014_10_12_000000_create_users_table
Migrated: 2014_10_12_100000_create_password_resets_table
Migrated: 2015_10_03_141020_create_issues_table
Seeded: UserTableSeeder
```

Done! Vậy là chúng ta đã tạo ra được dữ liệu giả bằng việc sử dụng model factory. Hãy kiểm tra database và bạn sẽ thấy có dữ liệu trong đó.

### Testing sử dụng Laravel Model Factories
Cái lợi của việc sử dụng model factories là chúng ta cũng có thể sử dụng cho mục đích kiểm thử. Trước tiên, hãy tạo mới test cho `Issues`:

```
php artisan make:test IssuesTest
```

Sửa file `tests/IssuesTest.php`:

```php
<?php

use Illuminate\Foundation\Testing\WithoutMiddleware;
use Illuminate\Foundation\Testing\DatabaseMigrations;
use Illuminate\Foundation\Testing\DatabaseTransactions;

class IssuesTest extends TestCase
{
  use DatabaseTransactions;

  public function testIssueCreation()
  {
    factory(App\Issues::class)->create([
      'subject' => 'NAIL POPS!'
    ]);

    $this->seeInDatabase('issues', ['subject' => 'NAIL POPS!']);
  }
}
```

Ở đây chúng ta sử dụng `DatabaseTransactions` cho mỗi 1 transaction. Bên trong hàm `testIssueCreation()` sử dụng model factory. Phương thức `create` sẽ truyền vào 1 mảng với tên của column và 1 giá trị mặc định.

Phương thức `seeInDatabase` sẽ dùng để kiểm thử và khi kết quả trả về màu xanh thì có nghĩa là dữ liệu đã được tồn tại trong DB.

Như vậy, chúng ta đã biết cách sử dụng model factories cho việc seeding và testing. Ngoài ra nó còn có 1 số chức năng hữu ích khác nữa, hãy thử tìm hiểu thêm nhé.

### Một số chức năng khác của Model Factories
Trong quá trình chuẩn bị họp với LEMON, bạn chợt nhận ra thiếu sót trong thiết lập ban đầu. Bạn quên thêm cách để đánh dấu các issues đã được xử lý. Điều này thật làm bạn bối rối...

#### Multiple Factory Types
Sau khi thêm mới 1 migration cho trường "completed" thì model factory cần được điều chỉnh, nhưng sẽ là tiện lợi nếu có 1 tách biệt mà bạn có thể dễ dàng tạo các completed issue. Hãy định nghĩa factory như sau:

```php
$factory->defineAs(App\Issues::class, 'completed', function ($faker) use ($factory) {
  $issue = $factory->raw(App\Issues::class);

  return array_merge($issue, ['completed' => true]);
});
```

Factory này sử dụng "completed" cho tham số thứ 2 và hàm callback sẽ tạo mới issue và gộp các cột completed vào.

Nếu bạn muốn tạo luôn các 'completed' issue thì đơn giản là chạy:

```php
$completedIssue = factory(App\Issue::class, 'completed')->make();
```

#### Phương thức `make` và `create`
Trong ví dụ cuối, có thể các bạn để ý là chúng ta sử dụng phương thức `make()` thay vì `create()` như trước đó đã dùng. Dĩ nhiên là 2 phương thức này sẽ làm 2 việc khác nhau. `create` sẽ lưu dữ liệu vào database giống như Eloquent. Mặt khác `make` thì tạo ra model và không thêm mới. Nếu bạn quen với Eloquent thì nó có thể so sánh với:

```php
$issue = new \App\Issue(['subject' => 'My Subject']);
```

### Closing
Như bạn thấy đấy, model factories là 1 tính năng tuyệt vời trong Laravel giúp đơn giản hóa cả testing và database seeding.

Hi vọng bài viết hữu ích cho các bạn đang sử dụng Laravel.

Ngoài ra, các bạn có thể tham khảo [Laravel Model Factory States]() ở version v5.3.17.

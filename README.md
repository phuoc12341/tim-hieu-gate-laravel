# Phân Quyền với Gate trong Laravel

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Gates là các **Closures**  xác định user có quyền thực hiện một action nhất định hay không và thường được định nghĩa trong class **App\Providers\AuthServiceProvider** sử dụng **Gate facade**. Gates luôn nhận được một **user** làm first argument và có thể tùy chọn nhận thêm các đối số như **Eloquent model** có liên quan:   

```
/**
 * Register any authentication / authorization services.
 *
 * @return void
 */
public function boot()
{
    $this->registerPolicies();

    Gate::define('edit-settings', function ($user) {
        return $user->isAdmin;
    });

    Gate::define('update-post', function ($user, $post) {
        return $user->id === $post->user_id;
    });
}
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Gates cũng có thể được định nghĩa bằng style callback string theo kiểu **Class@method**, giống như **controllers**:

```
/**
 * Register any authentication / authorization services.
 *
 * @return void
 */
public function boot()
{
    $this->registerPolicies();

    Gate::define('update-post', 'App\Policies\PostPolicy@update');
}
```

## 1. Authorizing Actions

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Để authorize cho một action sử Gate, ta nên sử dụng các method: **allows** hoặc **denies**. Lưu ý rằng ta không bắt buộc phải truyền **user** hiện đã được xác thực **(authenticated user)** cho các method này. Laravel sẽ tự động lo việc truyền user này vào gate Closure:

```
if (Gate::allows('edit-settings')) {
    // The current user can edit settings
}

if (Gate::allows('update-post', $post)) {
    // The current user can update the post...
}

if (Gate::denies('update-post', $post)) {
    // The current user can't update the post...
}
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Nếu ta muốn xác định xem một user cụ thể nào đó có được phép thực hiện một action hay không, ta có thể sử dụng method **forUser** trên Gate facade:

```
if (Gate::forUser($user)->allows('update-post', $post)) {
    // The user can update the post...
}

if (Gate::forUser($user)->denies('update-post', $post)) {
    // The user can't update the post...
}
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Ta có thể ủy quyền cho nhiều actions cùng một lúc với các methods **any** hoặc **none**:

```
if (Gate::any(['update-post', 'delete-post'], $post)) {
    // The user can update or delete the post
}

if (Gate::none(['update-post', 'delete-post'], $post)) {
    // The user cannot update or delete the post
}
```


## 2. Authorizing Or Throwing Exceptions

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Nếu bạn **attempt** để authorize một action và tự động throw **Illuminate\Auth\Access\AuthorizationException** nếu user không được phép thực hiện action đã cho, ta có thể sử dụng action **Gate::authorize**. Instances của **AuthorizationException** sẽ được tự động converted thành **403 HTTP response**:

```
Gate::authorize('update-post', $post);

// The action is authorized...
```

## 3. Supplying Additional Context

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Các methods về gate cho phép các khả năng **(allows, denies, check, any, none, authorize, can, cannot)** và các directives Blade authorization **(@can, @cannot, @canany)** có thể nhận được một mảng làm **second argument**. Các phần tử mảng này được passed dưới dạng parameters đến gate và có thể được sử dụng cho context bổ sung khi đưa ra quyết định authorization:

```
Gate::define('create-post', function ($user, $category, $extraFlag) {
    return $category->group > 3 && $extraFlag === true;
});

if (Gate::check('create-post', [$category, $extraFlag])) {
    // The user can create the post...
}
```

## 4. Gate Responses

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Cho đến giờ, chúng ta chỉ kiểm tra các gates mà return các giá trị **boolean** đơn giản. Tuy nhiên, đôi khi ta có thể muốn return **response** chi tiết hơn, bao gồm **error message**. Để làm như vậy, ta có thể return **Illuminate\Auth\Access\Response** từ gate của mình:

```
use Illuminate\Auth\Access\Response;
use Illuminate\Support\Facades\Gate;

Gate::define('edit-settings', function ($user) {
    return $user->isAdmin
                ? Response::allow()
                : Response::deny('You must be a super administrator.');
});
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Khi trả về **authorization response** từ gate của mình, method **Gate::allow** vẫn sẽ trả về giá trị **boolean** đơn giản; tuy nhiên, ta có thể sử dụng method **Gate::inspect** để nhận được **authorization response** đầy đủ mà gate trả về:

```
$response = Gate::inspect('edit-settings', $post);

if ($response->allowed()) {
    // The action is authorized...
} else {
    echo $response->message();
}
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Tất nhiên, khi sử dụng method **Gate::authorize** để throw ra một **AuthorizationException** nếu action không được authorized, **error message** được cung cấp bởi authorization response sẽ được truyền đến **HTTP response**:

```
Gate::authorize('edit-settings', $post);

// The action is authorized...
```
## 5. Intercepting Gate Checks

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Đôi khi, ta có thể muốn grant cả các **abilities** cho một user cụ thể. Ta có thể sử dụng method **before** để xác định **callback** được chạy trước tất cả các kiểm tra authorization khác:

```
Gate::before(function ($user, $ability) {
    if ($user->isSuperAdmin()) {
        return true;
    }
});
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Nếu **before callback** trả về non-null result thì kết quả đó sẽ được coi là kết quả của việc check.


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Ta có thể sử dụng method after để xác định một callback sẽ được thực hiện sau khi tất cả các authorization checks khác:

```
Gate::after(function ($user, $ability, $result, $arguments) {
    if ($user->isSuperAdmin()) {
        return true;
    }
});
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Tương tự như before check, nếu **after callback** trả về kết quả non-null thì kết quả đó sẽ được coi là kết quả của việc check.

Tài liệu tham khảo: [https://laravel.com/docs/6.x/authorization](https://laravel.com/docs/6.x/authorization)



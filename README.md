# Vue Laravel Table
This is a pretty straight forward component to use for Laravel projects.<br>
It is only a single component that will render a table with pagination, limit, search and other stuff from a paginated model object.<br>
There are no CSS/SCSS included, this you can customize yourself in your own CSS files.

### Preparations
You need to have Vue installed and be able to register the component.<br>
You need to grab [Ziggy](https://github.com/tighten/ziggy) be able to use route names with Vue.<br>
You need to setup CSRF to work with Vue.<br>
You need to name your route where you grab the data.<br>
See example setup section for code examples.<br>

### Installation
`npm install git+https://github.com/kaizokupuffball/vue-laravel-table.git`

### Example setup
```html
<!-- app.blade -->
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>laravel-table</title>
    </head>

    <body>
        <div id="app">
            <vue-laravel-table 
                name="users"
                data-route="api.users"
                :limit="{ take: 25 }"
                :headers="[
                    { display: 'ID', column: 'id', orderable: true, searchable: false },
                    { display: 'Name', column: 'name', orderable: true, searchable: true },
                    { display: 'E-mail', column: 'email', orderable: false, searchable: true }
                ]"
                :crud-routes="{
                    create: { name: 'users.create', icon: 'fas fa-plus' },
                    show: { name: 'users.show', icon: 'fas fa-eye' },
                    edit: { name: 'users.edit', icon: 'fas fa-edit' },
                    destroy: { name: 'users.destroy', icon: 'fas fa-trash', bulk: true }, 
                }"
            />
        </div>
    </body>

    <!-- Ziggy routes directive -->
    @routes
    <script>window.Laravel = { csrfToken: '{{ csrf_token() }}' }</script>
    <script src="{{ asset('js/app.js') }}"></script>
</html>
```

```js
// app.js
import axios from 'axios'
import { createApp } from 'vue'
import VueLaravelTable from 'vue-laravel-table'

const app = createApp({
    components: {
        VueLaravelTable
    }
});

// Ziggy from routes directive in blade file
app.config.globalProperties.$route = window.route;

// Axios for requests
app.config.globalProperties.$axios = axios;

// CSRF token from blade file
app.config.globalProperties.$axios.defaults.headers.common['X-Requested-With'] = 'XMLHttpRequest';
app.config.globalProperties.$axios.defaults.headers.common['X-CSRF-TOKEN'] = window.Laravel.csrfToken;
app.mount('#app')
```

```php
// routes/api.php
<?php

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::get('/users', function (Request $request) {

    // The model you want to create a table of
    $result = User::select('id', 'name', 'email');
    
    // Check for search query
    if ($request->q) {
        $q = $request->q;
        $wheres = explode(',', $request->where);
        for($i = 0; $i < count($wheres); $i++) {
            ($i == 0) && $result = $result->where($wheres[$i], 'LIKE', '%'. $q .'%');
            ($i > 0) && $result = $result->orWhere($wheres[$i], 'LIKE', '%'. $q .'%');
        }
    }
    
    // Check for ordering & sorting
    ($request->sortBy) && $result = $result->orderBy($request->sortBy, $request->sortDirection);
    
    // Paginate by limit
    $result = $result->paginate($request->limit);
    
    // Return the paginated object in JSON format
    return response()->json($result);

})->name('api.users'); // We need names routes for the table to work
```

```php
// UserController.php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;

class UserController extends Controller
{
    ...
    
    /**
     * Remove the specified resource from storage.
     *
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function destroy($id)
    {
        // Delete model(s)
        // We use this method since we can do bulk deletions with the table generated
        User::findOrFail(explode(',', $id))->filter(function($model) {
            $model->delete();
        });
        
        // Return a message and a type (primary/danger/success)
        return response()->json([
            'message' => 'Model deletion successfull!',
            'type' => 'success'
        ]);
    }
}
```

### Options
`name` should be a string wich will be used to identify the table with a class.<br>
`data-route` should be a route name where you want to fetch the data.<br>
`:crud-routes` should be a object that contains objects. See example.<br>
`:headers` should be and array of objects. See example.<br>
`:limit` this is being reworked!<br>
`notificationTimeout` this is how long the deletion notifications will be displayed in ms. It is optional, default value is set to 500ms.<br>

### Rendered
If everything is setup correctly it should look something like this.
![Show off](https://i.imgur.com/KQNyDAi.gif)

### Other
You may also put the search/filtering in actions if you rather prefer that. That is totally up to you. Example is just to show how it can be done.
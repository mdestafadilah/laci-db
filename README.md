
LaciDB - Flat File JSON DBMS
======================================

[![Build Status](https://img.shields.io/travis/emsifa/laci-db.svg?style=flat-square)](https://travis-ci.org/emsifa/laci-db)
[![License](http://img.shields.io/:license-mit-blue.svg?style=flat-square)](http://doge.mit-license.org)

## Overview

LaciDB adalah flat file DBMS dengan format penyimpanan berupa JSON. Karena format JSON, LaciDB bersifat *schemaless* seperti hanya NoSQL lainnya. Sebuah record dapat memiliki kolom yang berbeda-beda.

Dalam LaciDB tidak ada istilah table, yang ada adalah collection. Collection pada LaciDB mewakili sebuah file yang menyimpan banyak records.

Nama 'Laci' sendiri diambil karena prosesnya seperti laci pada meja/lemari. Laci pada meja/lemari umumnya tidak membutuhkan kunci (autentikasi), cukup buka > ambil sesuatu dan|atau taruh sesuatu > tutup. Pada LaciDB pun seperti itu, setiap query akan membuka file > eksekusi query (select|insert|update|delete) > file ditutup.

LaciDB tidak cocok untuk:

* Menyimpan database dengan ukuran yang besar.
* Menyimpan database yang membutuhkan keamanan tingkat tinggi.

LaciDB dibuat untuk:

* Menangani data-data yang kecil seperti pengaturan, atau data-data kecil lain.
* Backup dan import data lebih mudah. Bagi hosting yang mendukung git, kamu cukup melakukan push untuk backup, dan pull kalau mau import dari komputer lokal ke komputer server (atau sebaliknya).

## Cara Kerja

TODOC

## Examples

#### Initialize

```php
use Emsifa\Laci\Collection;

$collection = new Collection(__DIR__.'/db/users.json');
```

#### Insert Data

```php
$user = $collection->insert([
    'name' => 'John Doe',
    'email' => 'johndoe@mail.com',
    'password' => password_hash('password', PASSWORD_BCRYPT)
]);
```

`$user` would be something like:

```php
[
    '_id' => '58745c13ad585',
    'name' => 'John Doe',
    'email' => 'johndoe@mail.com',
    'password' => '$2y$10$eMF03850wE6uII7UeujyjOU5Q2XLWz0QEZ1A9yiKPjbo3sA4qYh1m'
]
```

> '_id' is uniqid()

#### Find Single Record By ID

```php
$user = $collection->find('58745c13ad585');
```

#### Find One

```php
$user = $collection->where('email', 'johndoe@mail.com')->first();
```

#### Select All

```php
$data = $collection->all();
```

#### Update

```php
$collection->where('email', 'johndoe@mail.com')->update([
    'name' => 'John',
    'sex' => 'male'
]);
```

> Return value is count affected records

#### Delete

```php
$collection->where('email', 'johndoe@mail.com')->delete();
```

> Return value is count affected records

#### Multiple Inserts

```php
$bookCollection = new Collection('db/books.json');

$bookCollection->inserts([
    [
        'title' => 'Foobar',
        'published_at' => '2016-02-23',
        'author' => [
            'name' => 'John Doe',
            'email' => 'johndoe@mail.com'
        ],
        'star' => 3,
        'views' => 100
    ],
    [
        'title' => 'Bazqux',
        'published_at' => '2014-01-10',
        'author' => [
            'name' => 'Jane Doe',
            'email' => 'janedoe@mail.com'
        ],
        'star' => 5,
        'views' => 56
    ],
    [
        'title' => 'Lorem Ipsum',
        'published_at' => '2013-05-12',
        'author' => [
            'name' => 'Jane Doe',
            'email' => 'janedoe@mail.com'
        ],
        'star' => 4,
        'views' => 96
    ],
]);

```

#### Find Where

```php
// select * from book.json where author[name] = 'Jane Doe'
$bookCollection->where('author.name', 'Jane Doe')->get();

// select * from book.json where star > 3
$bookCollection->where('star', '>', 3)->get();

// select * from book.json where star > 3 AND author[name] = Jane Doe
$bookCollection->where('star', '>', 3)->where('author.name', 'Jane Doe')->get();
```

> Operator can be '=', '<', '<=', '>', '>=', 'in', 'not in', 'between', 'match'.

#### Implementing `OR` Using Filter

```php
$bookCollection->filter(function($row) {
    return $row['star'] > 3 OR $row['author.name'] == 'Jane Doe';
})->get();
```

> `$row['author.name']` is equivalent with `$row['author']['name']`

#### Select Specify Keys

```php
// select author, title from book.json where star > 3
$bookCollection->where('star', '>', 3)->get(['author.name', 'title']);
```

#### Select As

```php
// select author[name] as author_name, title from book.json where star > 3
$bookCollection->where('star', '>', 3)->get(['author.name:author_name', 'title']);
```

#### Mapping

```php
$bookCollection->map(function($row) {
    $row['score'] = $row['star'] + $row['views'];
    return $row;
})
->sortBy('score', 'desc')
->get();
```

#### Sorting

```php
// select * from book.json order by star asc
$bookCollection->sortBy('star')->get();

// select * from book.json order by star desc
$bookCollection->sortBy('star', 'desc')->get();

// custom sorting 
$bookCollection->sort(function($a, $b) {
    return $a['star'] < $b['star'] ? -1 : 1;
})->get();
```

#### Limit & Offset

```php
// select * from book.json offset 4
$bookCollection->skip(4)->get();

// select * from book.json limit 10 offset 4
$bookCollection->take(10, 4)->get();
```

#### Join

```php
$userCollection = new Collection('db/users.json');
$bookCollection = new Collection('db/books.json');

// get user with 'books'
$userCollection->withMany($bookCollection, 'books', 'author.email', '=', 'email')->get();

// get books with 'user'
$bookCollection->withOne($userCollection, 'user', 'email', '=', 'author.email')->get();
```

#### Map and Save

```php
$bookCollection->where('star', '>', 3)->map(function($row) {
    $row['star'] = $row['star'] += 2;
    return $row;
})->save();
```

#### Transaction

```php
$bookCollection->begin();

try {

    // insert, update, delete, etc 
    // will stored into variable (memory)

    $bookCollection->commit(); // until this

} catch(Exception $e) {

    $bookCollection->rollback();

}
```


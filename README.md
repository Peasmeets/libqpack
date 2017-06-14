# QPack-C
Fast and efficient data serializer for the C program language.

---------------------------------------
  * [Installation](#installation)
  * [Usage](#usage)
    * [Packer](#packer)
    * [Unpacker](#unpacker)
  * [API](#api)
    * [qp_packer_t](#qp_packer_t)
    * [qp_unpacker_t](#qp_unpacker_t)
    * [qp_res_t](#qp_res_t)

---------------------------------------

## Installation
Install debug or release version, in this example we will install the release version.
```
$ cd Release
```

Compile qpack
```
$ make all
```

Install qpack
```
$ sudo make install
```

> Note: run `sudo make uninstall` for removal.

## Usage
The following data structure will be packed and unpacked to illustrate how
qpack can be used:

```json
{
    "What is the answer to life the universe and everything?": 42
}
```
### Packer
First we start to pack the data.
```c
#include <qpack.h>
#include <assert.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <inttypes.h>

const char * q = "What is the answer to life the universe and everything?";
const int a = 42;

int main(void)
{
    qp_packer_t * packer = qp_packer_create(512);
    if (packer == NULL) {
        abort();
    }

    /* Open a map. */
    qp_add_map(&packer);

    /* Add a key. We will add the question. Note that the question will be
     * packed without the teminator (null) character. */
    qp_add_raw(packer, q, strlen(q)); /* add question as key */

    /* Add a value. QPack only supports signed intergers. */
    qp_add_int64(packer, (int64_t) a);

    /* Close the map. Note that in a map an even number of items must be placed
     * which represent key/value pairs. */
    qp_close_map(packer);

    /* Print the packed data */
    qp_packer_print(packer);

    /* cleanup the packer */
    qp_packer_destroy(packer);

    return 0;
}
```

### Unpacker
Now we create some code to unpack the data.
```c
void print_qa(const unsigned char * data, size_t len)
{
    qp_unpacker_t unpacker;
    qp_res_t * res;
    int rc;

    /* Initialize the unpacker */
    qp_unpacker_init(&unpacker, data, len);

    /* We can get the data from the unpacker in two ways. One is to step over
     * the data using qp_next() and the other is to unpack all to a qp_res_t
     * object. The last option is probably easier and is what we will use in
     * this example but note that it consumes more memory and is potentially
     * slower compared to qp_next() for some use cases. */

    res = qp_unpacker_res(&unpacker, &rc);
    if (rc) {
        fprintf(stderr, "error: %s\n", qp_strerror(rc));
    } else {
        /* Usually you should check your response not with assertions */
        assert (res->tp == QP_RES_MAP);
        assert (res->via.map->n == 1); /* one key/value pair */
        assert (res->via.map->keys[0].tp == QP_RES_STR);   /* key */
        assert (res->via.map->values[0].tp == QP_RES_INT64); /* val */

        printf(
            "Unpacked:\n"
            "  Question: %s\n"
            "  Answer  : %" PRId64 "\n",
            res->via.map->keys[0].via.str,
            res->via.map->values[0].via.int64);

        /* cleanup res */
        qp_res_destroy(res);
    }
}

/* Call the function from main (see packer example for full code) */
int main(void)
{
    // ... see packer example for full code

    /* Call print_qa() before destroying the packer using member property
     * qp_packer_t.buffer which contains the packer data and qp_packer_t.len
     * for the size of the data */
    print_qa(packer->buffer, packer->len);

    /* cleanup the packer */
    qp_packer_destroy(packer);

    return 0;
}
```

## API

### `qp_packer_t`
Object which is used to pack data.

*Public members*
- `unsigned char * qp_packer_t.buffer`: contains the packed data (readonly)
- `size_t qp_packer_t.len`: the length of the data (readonly)

#### `qp_packer_t * qp_packer_create(size_t alloc_size)`
Create and return a new packer instance. Argument `alloc_size` should be at
least greather than 0 since this value is used to allocate memory. When more
memory is required while packing data the size will grow in steps of
`alloc_size`.

#### `void qp_packer_destroy(qp_packer_t * packer)`
Cleaunup packer instance.

#### `int qp_add_raw(qp_packer_t * packer, const char * raw, size_t len)`
Add raw data to the packer.

Returns 0 if successful or `QP_ERR_ALLOC` in case more memory is required which
cannot be allocated.

Example:
```c
int rc;
const char * s = "some string";
/* We use strlen(s) so the string will be packed without the terminator (null)
 * character. */
if ((rc = qp_add_raw(packer, s, strlen(s)))) {
    fprintf(strerr, "error: %s\n", qp_strerror(rc));
}
```

#### `int qp_add_int64(qp_packer_t * packer, int64_t i)`
Add an integer value to the packer. Note that only signed integers are
supported by QPack so unsigned integers should be casted. This function
accepts only `int64_t` but tries to pack the value as small as possible.

Returns 0 if successful or `QP_ERR_ALLOC` in case more memory is required which
cannot be allocated.

#### `int qp_add_double(qp_packer_t * packer, double d)`
Add a double value to the packer.

Returns 0 if successful or `QP_ERR_ALLOC` in case more memory is required which
cannot be allocated.

#### `int qp_add_true(qp_packer_t * packer)`
Add boolean value `TRUE` to the packer.

Returns 0 if successful or `QP_ERR_ALLOC` in case more memory is required which
cannot be allocated.

#### `int qp_add_false(qp_packer_t * packer)`
Add boolean value `FALSE` to the packer.

Returns 0 if successful or `QP_ERR_ALLOC` in case more memory is required which
cannot be allocated.

#### `int qp_add_null(qp_packer_t * packer)`
Add `NULL` to the packer.

Returns 0 if successful or `QP_ERR_ALLOC` in case more memory is required which
cannot be allocated.

#### `int qp_add_array(qp_packer_t ** packaddr)`
Add an `ARRAY` to the packer. Do not forget to close the array using
`qp_close_array()` although closing is not required by qpack in case
no other values need to be added after the array.

>Note: this function needs a pointer-pointer to a packer since the function
>might re-allocate memory for the packer.

Returns 0 if successful or `QP_ERR_ALLOC` in case more memory is required which
cannot be allocated.

#### `int qp_add_map(qp_packer_t ** packaddr)`
Add an `MAP` to the packer for storing key/value pairs. Do not forget to close
the map using `qp_close_map()` although closing is not required by qpack in case
no other values need to be added after the map. Always add an even number of
items which represent key/value pairs.

>Note: this function needs a pointer-pointer to a packer since the function
>might re-allocate memory for the packer.

Returns 0 if successful or `QP_ERR_ALLOC` in case more memory is required which
cannot be allocated.

#### `int qp_close_array(qp_packer_t * packer)`
Close an array.

Returns 0 if successful or `QP_ERR_ALLOC` in case more memory is required which
cannot be allocated. `QP_ERR_NO_OPEN_ARRAY` can be returned in case no *open*
array was found.

#### `int qp_close_map(qp_packer_t * packer)`
Close a map.

Returns 0 if successful or `QP_ERR_ALLOC` in case more memory is required which
cannot be allocated. `QP_ERR_NO_OPEN_MAP` can be returned in case no *open*
map was found or `QP_ERR_MISSING_VALUE` is returned (and the map will *not* be
closed) if the map contains a key without value.

#### `void qp_packer_print(qp_packer_t * packer)`
Macro function for printing packer content to stdout.

### `qp_unpacker_t`
Object which is used to unpack data.

*Public members*
- `const unsigned char * qp_unpacker_t.pt`:
  pointer to the current position in the data
- `const unsigned char * qp_unpacker_t.start`:
  pointer to data start (readonly)
- `const unsigned char * qp_unpacker_t.end`:
  pointer to data end (readonly)

#### `void qp_unpacker_init(qp_unpacker_t * unpacker, const unsigned char * pt, size_t len)`
Initialize an `qp_unpacker_t` instance. No additional clean is required.

#### `qp_types_t qp_next(qp_unpacker_t * unpacker, qp_obj_t * qp_obj)`
Used walk over an unpacker instance step-by-step. After calling the function,
`qp_obj` points to the current item in the unpacker. `qp_obj.tp` is always equal
to the return value of this function but note that `qp_obj` is allowed to be
`NULL` in which case you only have the return value.

Example (this code can replace the `print_qa()` code in
the [unpacker example](#unpacker):
```c
qp_obj_t question, answer;

assert (qp_is_map(qp_next(&unpacker, NULL)));
assert (qp_is_raw(qp_next(&unpacker, &question)));
assert (qp_is_int(qp_next(&unpacker, &answer)));

printf(
    "Unpacked:\n"
    "  Question: %.*s\n"
    "  Answer  : %" PRId64 "\n",
    (int) question.len, question.via.raw,
    answer.via.int64);
```
#### `qp_res_t * qp_unpacker_res(qp_unpacker_t * unpacker, int * rc)`
Returns a pointer to a [qp_res_t](#qp_res_t) instance which is created from an
unpacker instance or `NULL` in case of an error. Argument `rc` will be 0 when
successful or `QP_ERR_ALLOC` in case required memory can not be allocated.
The value of `rc` can also be `QP_ERR_CORRUPT` when the unpacker contains data
which cannot be unpacked. `rc` is allowed to be `NULL`.

#### `void qp_unpacker_print(qp_unpacker_t * unpacker)`
Macro function for printing unpacker content to stdout.

### `qp_res_t`
Object which contains unpacked data. An instance of `qp_res_t` is created with
the [qp_unpacker_t](#qp_unpacker_t) function `qp_unpacker_res()`.

*Public members*
- `qp_res_tp qp_res_t.tp`: Type of the result (readonly)
  - QP_RES_MAP
  - QP_RES_ARRAY
  - QP_RES_INT64
  - QP_RES_REAL
  - QP_RES_STR
  - QP_RES_BOOL
  - QP_RES_NULL
- `qp_res_via_t qp_res_t.via`: Union containing the actual data (readonly)

#### `void qp_res_destroy(qp_res_t * res)`
Cleanup result instance.

### Miscellaneous functions
#### `const char * qp_strerror(int err_code)`
Returns a string representation for an error code.

#### `const char * qp_version(void)`
Returns a string representation the version of qpack.

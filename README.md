# CC - The updated C language

CC is my 'sequel' to the C programming language. Below are listed all of the changes made to CC.

## 1. New types

CC defines some new types:

    int8, int16, int32, int64, uint8, uint16, uint32, uint64, uint
    mptr

The most interesting new type is mptr. To explain why you want an mptr (machine ptr), look at this example from C:
  
    struct wrapper{
      void* object_array;
      size_t object_size;
      unsigned length;
    }

    void* get_object(struct wrapper* wrapper, unsigned n){
      return ((uint8_t*)wrapper->object_array) + n*wrapper->object_size;
    }

In this code, you need to cast `wrapper->object_array` to be a `uint8_t*` in order to perform pointer arithmetic on it.
One fix could be to define `object_array` as a `uint8_t*` rather than a `void*` however this is confusing because this
implies the data pointed to by `wrapper->object_array` is of the type `uint8_t` when in reality it is unknown. So if you
want to do pointer arithmetic on `void*` you are out of luck (unless you want to violate standard C which most modern
C compilers allow anyways). My solution to this problem is to define a new type which, like void, doesn't describe the data it points at but also allows pointer math, essentially aliasing the data to be an integer.

## 2. Stronger Preprocessor

CC's preprocessor is stronger than C's. Let's take a look at an example where C's preprocessor is a bit lackluster:

    typedef struct {
      int a, b;
      char c;
    } mystruct;

    #ifdef DEBUG
    void log_memory_allocation(void *mem, char *file, int line){
      printf("memory allocated at %p in file %s at line %d\n", mem, file, line);
    }
    #endif

    mystruct *mystruct_create(){
      #ifdef DEBUG
      void* mem = malloc(sizeof(mystruct));
      log_memory_allocation(mem, __FILE__, __LINE__);
      #else
      return malloc(sizeof(mystruct));
      #endif
    }

    int main(){
      mystruct *s1, *s2, *s3;
      
      s1 = mystruct_create();
      s2 = mystruct_create();
      s3 = mystruct_create();

      return 0;
    }
    
The `log_memory_allocation` function is helpful for keeping track of mallocs. The problem with this set, however, is that
typically malloc is wrapped in a `struct_create()` function. So the information from the `__FILE__` and `__LINE__` 
macros isn't super helpful. One possible solution would be:

    .
    .
    .

    mystruct *mystruct_create(){
      return malloc(sizeof(mystruct));
    }


    int main(){
      mystruct *s1, *s2, *s3;

      s1 = mystruct_create();
      #ifdef DEBUG
      log_memory_allocation(s1, __FILE__, __LINE__);
      #endif

      s2 = mystruct_create();
      #ifdef DEBUG
      log_memory_allocation(s2, __FILE__, __LINE__);
      #endif

      s3 = mystruct_create();
      #ifdef DEBUG
      log_memory_allocation(s3, __FILE__, __LINE__);
      #endif

      return 0;
    }

While this 'works', it's an awful solution. It's extremely cumbersome and ugly (not to mention that `__LINE__` will now 
be incorrect because the log is on the next line! I suppose you could put `__LINE__ - 1` but that makes this even worse!)
Ok, so how can we fix this? We can use the new after statement!

    #ifdef DEBUG
    #after (mystruct_create, ()) log_memory_allocation(@after.result, __FILE__, __LINE__);
      
    void log_memory_allocation(void *mem, char *file, int line){
      printf("memory allocated at %p in file %s at line %d\n", mem, file, line);
    }
    #endif

    typedef struct {
      int a, b;
      char c;
    } mystruct;

    mystruct *mystruct_create(){
      return malloc(sizeof(mystruct));
    }

    int main(){
      mystruct *s1, *s2, *s3;
      
      s1 = mystruct_create();
      s2 = mystruct_create();
      s3 = mystruct_create();

      return 0;
    }

This is equivalent to the C code from the previous example. So how does it work? The new `#after` directive lets you 
run code after other code dynamically from the preprocessor. The first argument to `#after` is the object to attach code 
to. The second argument is the operation to attach it to, in this instance it's the `()` operation for function calls.
Finally, you specify the code you want to run after. You can pass in the result of the operation using the `@result` 
macro option. The `@` syntax is new, it specifies special options that come with macros. We'll discuss more about them 
later. Naturally, there is also a `#before` macro which attaches code before the operation rather than after. For now, 
I want to showcase something interesting that can be done with this new preprocessor:

    #after  (object*, =:) object_reference_up(@after.instance);
    #before (object*, :=) object_reference_down(@before.instance);

    typedef struct {
      int reference_counter;
      int data;
    } object;

    object *object_create(){
      object *ret = malloc(sizeof *ret);
      ret->reference_counter = 0;
      return ret;
    }

    void object_destroy(object* o){
      free(o);
    }

    void object_reference_up(object* o){
      ++o->reference_counter;
    }

    void object_reference_down(object* o){
      if(!--object->reference_counter){
        object_destroy(object);
      }
    }

    int main(){
      object* myobject = object_create();
      object* another_pointer = myobject;
      myobject = NULL;
      printf("%d\n", another_pointer->reference_counter);
      another_object = NULL;
    }

Using only macors, we have developed a reference counting self collecting object! Awesome! Let's explain some of the
gritty details. The first new thing is that there is a type instead of a specific instance in the first argument to
after. This just means that the directive will match with all instances of that type instead of a specific variable.
Next we see `=:`. This sign will match whenever an `object*` is used as an rvalue, denoted by the colon on the right.
Next we have `@after.instance`, this evaluates to the matched object, not the result of the operation as we saw in the
previous example. Everything is similar in the `#before` statement.

Here is a summary of all the new preprocessor features:

    #after (<instance_name or type_name>, <@after.operation>) {}
      @after.pre_increment - ++x
      @after.post_increment - x++
      @after.pre_decrement - --x
      @after.post_decrement - x--
      @after.compare                      ==
      @after.lcompare                     :==
      @after.rcompare                     ==:
      @after.lvalue                       :=
      @after.rvalue                       =:
      @after.multiply
      @after.add                          +
      @after.divide                       /
      @after.subtract                     -
      @after.indirection                  ->
      @after.dereference                  
      @after.and                          &
      @after.or                           |
      @after.xor                          ^
      @after.flip                         ~
      @after.not                          !
      @after.lshift                       <<  
      @after.rshift                       >>
      @after.call                         ()
      @after.subscript                    []

      @after.result: result of operation
      @after.instance: if instance is used as first argument, it will be replaced here. If it's
                       a type name, it will be the matched instance of the type name

    #before (<instance_name or type_name>, <@before.operation>) {}
      @before.pre_increment - ++x
      @before.post_increment - x++
      @before.pre_decrement - --x
      @before.post_decrement - x--
      @before.compare                      ==
      @before.lcompare                     :==
      @before.rcompare                     ==:
      @before.lvalue                       :=
      @before.rvalue                       =:
      @before.multiply
      @before.add                          +
      @before.divide                       /
      @before.subtract                     -
      @before.indirection                  ->
      @before.dereference                  
      @before.and                          &
      @before.or                           |
      @before.xor                          ^
      @before.flip                         ~
      @before.not                          !
      @before.lshift                       <<  
      @before.rshift                       >>
      @before.call                         ()
      @before.subscript                    []

      @before.instance: if instance is used as first argument, it will be replaced here. If it's
                       a type name, it will be the matched instance of the type name

## 3. Some new operators and syntax

For readability, some of the classic logical operators are getting some new paint:

    old:    new:
    ==      equals
    !       not
    ||      or
    &&      and

NOTE: you may still use the old C style operators.

Forever loops are a bit unsightly in C, so CC makes it a bit better:

    forever{              for(;;){
                <==>    
    }                     }

Unnamed functions can be defined on the fly, which is really nice:

    int (*funcptr)(int a, int b) = (int a, int b) {
      return a + b * a - b;
    }

    int (*foo)(int, int) = funcptr;
    funcptr = NULL;
    foo(2, 3);


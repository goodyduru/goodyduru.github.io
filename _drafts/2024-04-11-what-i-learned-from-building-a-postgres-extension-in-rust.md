---
layout: post
title: "What I Learned From Building A Postgres Extension In Rust"
date: 2024-04-11
categories: database
---

This is a list of things I learned from building the [postgres-redis](https://github.com/systemEng-Learning/postgres-redis) extension for Postgres in Rust. A retrospective design and implementation article, this is not. I wrote that up [here]().

Building this extension opened my eyes towards some of Postgres design details and Rust tricks. The things I learned might be normal and widely known, but I learned them on this project. Here's what I learned.

### The Mundanity Of Excellence
Take a look at this piece of code

```c
/*
	 * Loop until we've processed the proper number of tuples from the plan.
	 */
	for (;;)
	{
		/* Reset the per-output-tuple exprcontext */
		ResetPerTupleExprContext(estate);

		/*
		 * Execute the plan and obtain a tuple
		 */
		slot = ExecProcNode(planstate);

		/*
		 * if the tuple is null, then we assume there is nothing more to
		 * process so we just end the loop...
		 */
		if (TupIsNull(slot))
			break;

		/*
		 * If we have a junk filter, then project a new tuple with the junk
		 * removed.
		 *
		 * Store this new "clean" tuple in the junkfilter's resultSlot.
		 * (Formerly, we stored it back over the "dirty" tuple, which is WRONG
		 * because that tuple slot has the wrong descriptor.)
		 */
		if (estate->es_junkFilter != NULL)
			slot = ExecFilterJunk(estate->es_junkFilter, slot);

		/*
		 * If we are supposed to send the tuple somewhere, do so. (In
		 * practice, this is probably always the case at this point.)
		 */
		if (sendTuples)
		{
			/*
			 * If we are not able to send the tuple, we assume the destination
			 * has closed and no more tuples can be sent. If that's the case,
			 * end the loop.
			 */
			if (!dest->receiveSlot(slot, dest))
				break;
		}

		/*
		 * Count tuples processed, if this is a SELECT.  (For other operation
		 * types, the ModifyTable plan node must count the appropriate
		 * events.)
		 */
		if (operation == CMD_SELECT)
			(estate->es_processed)++;

		/*
		 * check our tuple count.. if we've processed the proper number then
		 * quit, else loop again and process more tuples.  Zero numberTuples
		 * means no limit.
		 */
		current_tuple_count++;
		if (numberTuples && numberTuples == current_tuple_count)
			break;
	}
```
Pretty normal, no clever tricks, just straightforward C code. This piece of code [taken](https://github.com/postgres/postgres/blob/b1b13d2b524e64e3bf3538441366bdc8f6d3beda/src/backend/executor/execMain.c#L1638) from Postgres is responsible for inserting/updating/deleting/selecting tuples (rows) from a table. It's pretty amazing how simple it is, and yet almost all select/update/delete/insert query you run against your Postgres table executes this code. It wasn't just this area, other parts of the Postgres codebase was like this. Simple pieces of code that combine together to create an excellent, reliable, sophisticated and amazing software. It reminds me of Da Vinci quote: Simplicity is the ultimate sophistication.

### Rust Pointers
Working on this project introduced me to using pointers in Rust lots of time. It turns out that in an unsafe context, Rust pointers behave so much like C. While the basic pointer stuff like referencing and dereferencing pointers are covered well by the Rust [book](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer), there are two things I learnt about Rust pointers when building this extension. Here they are:

* Pointer arithmetic: Imagine you have a list of objects and you have the pointer to the first item, how do you get to the nth item. In C, you'd probably do this:

```c
struct object* get_nth(struct object* list, int n) {
    struct object* item = list + n;
    return item;
}
```

I needed to do something like this when porting this [function](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer) from C to Rust. Thankfully, Rust has a really handy function for that. It's called [offset](https://doc.rust-lang.org/std/primitive.pointer.html#method.offset) and it worked well. Here's how the above code will look in Rust:

```rust
unsafe fn get_nth(list: *mut object, n: isize) -> *mut object {
    let item = list.offset(n);
    item
}
```

That's it!!! Really straightforward and dangerous 🙃. 

* Easily cast a struct variable to another: Rust protects the programmer a lot. More importantly, it protects the programmer from themselves. Let's say you want to convert a variable from one struct type to another struct type. In Rust, we want to achieve something like this:

```rust
struct Person {
    firstname: String,
    lastname: String,
}

struct User {
    firstname: String,
    lastname: String,
}

fn convert_person_to_user(x: Person) {
    let u = x as User;
    println!("Firstname: {}, Lastname: {}", u.firstname, u.lastname);
}
```

When you run this code, the Rust compiler will complain. It will complain irrespective of the fact that both structs are similar except for their names. This is a good thing, but it might be a bit inconvenient. There are other ways you could achieve this, rather than using straight up casting. But what if you are working with library code or C code and you need to cast. We can achieve it with pointers. Here's the above function rewritten to use pointers.

```rust 
unsafe fn convert_to_user(mut x: Person) {
    let p = &mut x as *mut Person;
    let u = p.cast::<User>();
    println!("{}, {}", (*u).firstname, (*u).lastname);
}
```

It runs without any complaint. It isn't good Rust code, but sometimes you need something to just work.

I won't advise anyone to use the above methods to solve problems, but it could happen that you need them. It happened that I needed them, and it was really worthwhile discovering and using them.

Speaking of casting....

### Adding extra fields to a struct
Let's imagine you're using a todo library that works with only one task. When you add a task, it throws away the previous task

```rust
#[derive(Clone)]
struct Task {
    content: String,
    is_done: bool
}

#[repr(C)]
struct Todo {
    add: unsafe fn(self_: *mut Todo, task: Task),
    current_task: Option<Task>,
}

unsafe fn add_task(todo: *mut Todo, task: Task) {
    (*todo).current_task = Some(task);
}

fn add_tasks(todo: *mut Todo) {
    let task1 = Task{content: String::from("Write"), is_done: false};
    let task2 = Task{content: String::from("Read"), is_done: true};
    unsafe {
        ((*todo).add)(todo, task1);
        ((*todo).add)(todo, task2);
    }
}
```

You'd use the above library like this

```rust
fn create_todo() {
    let mut todo = Todo {add: add_task, current_task: None};
    let todo_ptr = &mut todo as *mut Todo;
    add_tasks(todo_ptr);
}
```

It works well, but what if the one task at a time is too limited for you? What if you want to keep track of previous tasks? The library code is pretty fixed and expect a certain type of data. Is it possible to satisfy the library and satisfy your extra needs? It turns out that with a bit of casting trick, you could achieve both. Here's how?

```rust
#[repr(C)]
struct CustomTodo {
    add: unsafe fn(self_: *mut Todo, task: Task),
    current_task: Option<Task>,
    previous_tasks: Vec<Task>,
}

unsafe fn custom_add_task(todo: *mut Todo, task: Task) {
    let custom_todo = todo as *mut CustomTodo;
    if (*custom_todo).current_task.is_some() {
        let prev = (*custom_todo).current_task.as_ref().unwrap().clone();
        (*custom_todo).previous_tasks.push(prev);
    }
    (*custom_todo).current_task = Some(task);
}

fn create_todo() {
    let mut custom_todo = CustomTodo{add: custom_add_task, current_task: None, previous_tasks: vec![]};
    let custom_todo_ptr = &mut custom_todo as *mut CustomTodo;
    unsafe {
        let custom_todo_ptr = custom_todo_ptr as *mut Todo;
        add_tasks(custom_todo_ptr);
    }
    for task in &(custom_todo.previous_tasks) {
        println!("Previous tasks\tContent: {}, is_done: {}", task.content, task.is_done);
    }
}
```

The above code shows how we've casting to achieve both our needs. To pull this off, we needed a little help from the library code. Notice the [`repr`](https://doc.rust-lang.org/reference/type-layout.html#the-c-representation) attribute on both `Todo` and `CustomTodo`, that tells the Rust compiler to treat both like a C struct. The attribute ensures that the order of the fields isn't changed by the compiler. If the library hadn't added the attribute, a segmentation fault would have occured.

This is struct manipulation is dangerous and shouldn't be used in normal Rust code. They are ways to achieve the above intent without meddling in unsafe territory. With this warning, it can come in handy especially when dealing with C code.

### So Many Kinds Of Operators
Take a look at [this](https://github.com/postgres/postgres/blob/3741f2a09d5205ec32bd8af5d1f397e08995932b/src/include/catalog/pg_operator.dat#L100). Postgres has lots of operators. Initially I thought the `=` sign would only have one constant. I could not have been more mistaken. There are Int4EqualOperator, TextEqualOperator, NameEqualTextOperator, BooleanEqualOperator, TIDEqualOperator, BpcharEqualOperator, ByteaEqualOperator, and this is just for the equal sign. Other operators have their own different kinds, with specific oids for each kind.

It made our where clause handler a bit more complex but I can't complain. I'm curious about the reason for this though. Please if you know, I'd love to hear about it :-).


### Conclusion
This lessons might not be special to you, but they are to me. Working on this project has made me a better programmer. 

It's ironic that I started with praises about the simplicity of Postgres to then end with some clever tricks. I think the biggest lesson here is to strive for simplicity but you never know when a clever trick could come in handy. Want to know where I first found out about the struct adapting trick, [Postgres](https://github.com/postgres/postgres/blob/b1b13d2b524e64e3bf3538441366bdc8f6d3beda/src/backend/access/common/printtup.c#L72).



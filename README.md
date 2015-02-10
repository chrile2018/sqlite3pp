sqlite3pp
=========

<i>NEWS - With the latest updates, sqlite3pp became boost-free. You don't have to use boost to use sqlite3pp any more. If you want the boost friendly version, the files are in the boost_src directory. But, I highly recommend you use new version if you are using c++11 or later.</i>

It makes SQLite3 API more friendly to C++ users. It supports almost all of SQLite3 features using C++ classes such as database, command, query, and transaction. The query class supports iterator concept for fetching records.

With ext::function class, it's also easy to write the sqlite3's functions and aggregations.

# Usage

## database
```cpp
sqlite3pp::database db("test.db");
db.execute("INSERT INTO contacts (name, phone) VALUES ('Mike', '555-1234')");
```

## command
```cpp
sqlite3pp::command cmd(
  db, "INSERT INTO contacts (name, phone) VALUES (?, ?)");
cmd.binder() << "Mike" << "555-1234";
cmd.execute();
```

```cpp
sqlite3pp::command cmd(db, "INSERT INTO contacts (name, phone) VALUES (?, ?)");
cmd.bind(1, "Mike");
cmd.bind(2, "555-1234");
cmd.execute();
```

```cpp
sqlite3pp::command cmd(
  db, "INSERT INTO contacts (name, phone) VALUES (?100, ?101)");
cmd.bind(100, "Mike");
cmd.bind(101, "555-1234");
cmd.execute();
```

```cpp
sqlite3pp::command cmd(
  db, "INSERT INTO contacts (name, phone) VALUES (:user, :phone)");
cmd.bind(":user", "Mike");
cmd.bind(":phone", "555-1234");
cmd.execute();
```

## transaction

```cpp
sqlite3pp::transaction xct(db);
{
  sqlite3pp::command cmd(
    db, "INSERT INTO contacts (name, phone) VALUES (:user, :phone)");
  cmd.bind(":user", "Mike");
  cmd.bind(":phone", "555-1234");
  cmd.execute();
}
xct.rollback();
```

## query

```cpp
sqlite3pp::query qry(db, "SELECT id, name, phone FROM contacts");

for (int i = 0; i < qry.column_count(); ++i) {
  cout << qry.column_name(i) << "\t";
}
```

```cpp
for (sqlite3pp::query::iterator i = qry.begin(); i != qry.end(); ++i) {
  for (int j = 0; j < qry.column_count(); ++j) {
    cout << (*i).get<char const*>(j) << "\t";
  }
  cout << endl;
}
```

```cpp
for (sqlite3pp::query::iterator i = qry.begin(); i != qry.end(); ++i) {
  int id;
  char const* name, *phone;
  std::tie(id, name, phone) =
    (*i).get_columns<int, char const*, char const*>(0, 1, 2);
  cout << id << "\t" << name << "\t" << phone << endl;
}
```

```cpp
for (sqlite3pp::query::iterator i = qry.begin(); i != qry.end(); ++i) {
  string name, phone;
  (*i).getter() >> sqlite3pp::ignore >> name >> phone;
  cout << "\t" << name << "\t" << phone << endl;
}
```

## attach

```cpp
sqlite3pp::database db("foods.db");b
db.attach("test.db", "test");

sqlite3pp::query qry(
  db,
  "SELECT epi.* FROM episodes epi, test.contacts con WHERE epi.id = con.id");
```

## callback

```cpp
struct rollback_handler
{
  void operator()() {
    cout << "handle_rollback" << endl;
  }
};

sqlite3pp::database db("test.db");

db.set_commit_handler([]{ cout << "handle_commit\n"; return 0; });
db.set_rollback_handler(rollback_handler());
```

```cpp
int handle_authorize(int evcode, char const* p1, char const* p2,
                     char const* dbname, char const* tvname) {
  cout << "handle_authorize(" << evcode << ")" << endl;
  return 0;
}

db.set_authorize_handler(&handle_authorize);
```

```cpp
struct handler
{
  handler() : cnt_(0) {}

  void handle_update(int opcode, char const* dbname,
                     char const* tablename, int64_t rowid) {
    cout << "handle_update(" << opcode << ", " << dbname << ", "
         << tablename << ", " << rowid << ") - " << cnt_++ << endl;
  }
  int cnt_;
};

using namespace std::placeholders;

db.set_update_handler(std::bind(&handler::handle_update, &h, _1, _2, _3, _4));
```

## function

```cpp
int test0()
{
  return 100;
}

sqlite3pp::database db("test.db");
sqlite3pp::ext::function func(db);

func.create<int ()>("test0", &test0);
```

```cpp
void test1(sqlite3pp::ext::context& ctx)
{
  ctx.result(200);
}

void test2(sqlite3pp::ext::context& ctx)
{
  string args = ctx.get<string>(0);
  ctx.result(args);
}

void test3(sqlite3pp::ext::context& ctx)
{
  ctx.result_copy(0);
}

func.create("test1", &test1);
func.create("test2", &test2, 1);
func.create("test3", &test3, 1);
```

```cpp
func.create<int ()>("test4", []{ return 500; });
```

```cpp
string test5(string const& value)
{
  return value;
}

string test6(string const& s1, string const& s2, string const& s3)
{
  return s1 + s2 + s3;
}

func.create<int (int)>("test5", [](int i){ return i + 10000; });
func.create<string (string, string, string)>("test6", &test6);
```

```cpp
sqlite3pp::query qry(
  db,
  "SELECT test0(), test1(), test2('x'), test3('y'), test4(), test5(10), "
  "test6('a', 'b', 'c')");
```

## aggregate

```cpp
void step(sqlite3pp::ext::context& c)
{
  int* sum = (int*) c.aggregate_data(sizeof(int));

  *sum += c.get<int>(0);
}
void finalize(sqlite3pp::ext::context& c)
{
  int* sum = (int*) c.aggregate_data(sizeof(int));
  c.result(*sum);
}

sqlite3pp::database db("foods.db");
sqlite3pp::ext::aggregate aggr(db);

aggr.create("aggr0", &step, &finalize);
```

```cpp
struct mycnt
{
  void step() {
    ++n_;
  }
  int finish() {
    return n_;
  }
  int n_;
};

aggr.create<mycnt>("aggr1");
```

```cpp
struct strcnt
{
  void step(string const& s) {
    s_ += s;
  }
  int finish() {
    return s_.size();
  }
  string s_;
};

struct plussum
{
  void step(int n1, int n2) {
    n_ += n1 + n2;
  }
  int finish() {
    return n_;
  }
  int n_;
};

aggr.create<strcnt, string>("aggr2");
aggr.create<plussum, int, int>("aggr3");
```

```cpp
sqlite3pp::query qry(
  db,
  "SELECT aggr0(id), aggr1(type_id), aggr2(name), aggr3(id, type_id) "
  "FROM foods");
```



# See also
* http://www.sqlite.org/
* https://code.google.com/p/sqlite3pp/ 
* http://ideathinking.com/2015/02/using-variadic-templates-with-different-parameter-types.html
* [c-of-day-43-sqlite3-c-wrapper-1](http://idea-thinking.blogspot.com/2007/09/c-of-day-43-sqlite3-c-wrapper-1.html) (Korean)
* [c-of-day-44-sqlite3-c-wrapper-2](http://idea-thinking.blogspot.com/2007/09/c-of-day-44-sqlite3-c-wrapper-2.html) (Korean)
* [c-of-day-45-sqlite3-c-wrapper-3](http://idea-thinking.blogspot.com/2007/09/c-of-day-45-sqlite3-c-wrapper-3.html) (Korean)

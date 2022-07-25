## How Apollo Cache works

- First cache entry (When query all notes)

```
cache["notes:categoryId-1] = [
	{id: "1", content: "test01"},
	{id: "2", content: "test02"}
]
```

- Second cache entry (When query a single note)

```
cache["note:id-1"] = {
	id: "1",
	content: "test01",
	category:{
		"label": "tests"
	}
}
```

- `cache` will be structured as two objects with duplicated values. 1. This takes more memories. 2. Data inconsistency problem: If we update `note:id-1`, the first object `notes:categoryId-1` wouldn't be updated.

```
{
	"notes:categoryId-1": [
		{id: "1", content: "test01"},
		{id: "2", content: "test02"}
	],
	"note:id-1": {
		 id: "1",
		 content: "test01",
		 category: {
			{
				 label: "tests"
			}
		 }
	 }
}
```

### Solution

- Create an extra `cache` with references point to the object. Structured as [type:ID]:

```
cache["note:1"] = {id: "1", content: "test01"};
cache["note:2"] = {id: "1", content: "test02"};
```

- `notes:categoryId-1` became:

```
cache["notes:categoryId-1"] = [
	"note:1",
	"note:2"
]
```

- `note:id-1` can be represented using just the ID. When the second query is called, the new fields were added into the first cache with `note:1` as key.

```
cache["note:1"] = {
	...cache["note:1"],
	content: "test 01",
	category: {label: "tests"}
}

cache["note:id-1"] = "note:1";
```

#### Data normalisation: As a result, when the first query is updated, calling the second query using its reference ID `note:1` will return the updated data. Cache will be looking like:

```
{
	"note:1":  {id: "1", content: "test01", category: {
		label: "tests"
		}
	};
	"note:2":  {id: "1", content: "test02"};
	'notes:categoryId-1': ['note:1', 'note:2'],
	'note:id-1':'note:1'
}
```

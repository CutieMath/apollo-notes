# How Apollo Cache works

## Problem

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

## Solution

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

# How to update local cache

## Problem

- Data are normalised in the local Cache according above section.

## Solution

- Pass in returned elements ID when making mutation query. Apollo will automatically update the cached based on the ID:

```
const  [updateNote,  {  loading  }]  =  useMutation(gql`
	mutation UpdateNote($id: String!, $content: String!) {
		updateNote(id: $id, content: $content) {
			successful
			note {
				id
				content
			}
		}
	}
`);
```

Note:
Need to make sure that

- Return the changed obj
- Return the ID of the changed obj so Apollo knows what to update.

# How to delete data and display in UI in real time

## Problem

- UI doesn't update on delete when this mutation was used:

```
const [deleteNote] = useMutation (
	gql `
		mutation DeleteNote($noteId: String!){
			 successful
			 note {
				id
			 }
		}
	`
)
```

## Solution

- The reason this happened is bc Apollo didn't know we want to update the entire Notes object returned by this query:

```
const  NOTES_QUERY  =  gql`
	query GetAllNotes($categoryId: String) {
		notes(categoryId: $categoryId) {
			id
			content
			category {
				id
				label
			}
		}
	}
`;
```

- We need to return the full list of the updated notes. To make this query simpler, we can use a build in method in Apollo:
- Refetch option for mutation queries

```
const [deleteNote] = useMutation (
	gql `
		mutation DeleteNote($noteId: String!){
			 successful
			 note {
				id
			 }
		}
	`, {
		refetchQueries: ["GetAllNotes"]
	}
)
```

This will make two gql calls.

# How to make less GQL network calls for delete mutation

## Problem

- The above section will make two network calls.

## Solution

- Use the `cache.modify()` method to update the cache directly:
- Note:
  `cache.identify`, a utility function from Apollo was used here to check/create the ID of the deleted obj.

```
const  [deleteNote]  =  useMutation(
	gql`
		mutation DeleteNote($noteId: String!) {
			deleteNote(id: $noteId) {
				successful
				note {
					id
				}
			}
		}
	`,
	{
		update: (cache,  mutationResult) => {
			const  deletedNoteId  =  cache.identify(
				mutationResult.data?.deleteNote.note
			);
			cache.modify({
				fields: {
					notes: (existingNotes) => {
						return  existingNotes.filter((noteRef) => {
							return  cache.identify(noteRef) !==  deletedNoteId;
						});
					},
				},
			});
		},
	}
);
```

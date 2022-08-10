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

# How to optimise updates

## Problem

- The update function above only updates when a mutation query completes.

## Solution

- Use `optimisticResponse`.
- Assume the mutation query is successful:

```
optimisticResponse: (vars) => {
	return {
		deleteNote: {
			successful:  true,
			__typename:  "DeleteNoteResponse",
				note: {
					id:  vars.noteId,
					__typename:  "Note",
				},
			},
		};
	},
```

- Remember that Apollo cache use [type:ID] combined for the key. So the `__typename` must be specified.
- Apollo creates a new `optimistic cache` layer to handle this.

# How to evict cache

## Problem

- When a note is deleted from the UI, it's deleted from the RootQuery, but not from the cache.

## Solution

- Use `cache.evict()` to delete the cache completely
- This is to make sure other parts of the UI is up to date.

```
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
	cache.evict({ id:  deletedNoteId });
},
```

# How to implement LoadMore

## Problem

- When data became infinitely large, it's not practical to fetch all data and display in the front-end

## Solution

- Use `fetchMore` function from Apollo

1.  Add `$offset` and `$limit` parameters into query

```
query GetAllNotes($categoryId: String, $offset: Int, $limit: Int)
```

2. Add into they query's variable:

```
const { data, loading, error, fetchMore } =  useQuery(NOTES_QUERY, {
	variables: {
		categoryId: category,
		offset: 0,
		limit: 3,
	},
	fetchPolicy: "cache-and-network",
	errorPolicy: "all",
});
```

3. Add the `LoadMore` button. Use `fetchMore` from the Apollo client.

```
<UiLoadMoreButton
	onClick={() =>
		fetchMore({
			variables: {
				offset: data.notes.length,
			},
		})
	}
/>
```

4. Customise the behaviour of different fields in cache using TypePolicy

```
cache: new  InMemoryCache({
	typePolicies: {
		Query: {
			fields: {
				notes: {
					keyArgs: ["categoryId"],
					merge: (existingNotes  = [], incomingNotes) => {
						return [...existingNotes, ...incomingNotes];
					},
				},
			},
		},
	},
}),
```

# How to add custom fields into gql

## Problem

- An extra field is needed to determine UI. e.g. Selected box. This field does not exist in the backend.

## Solution

- Use `typePolicies` to store the state into Apollo central cache

1. Add the UI

```
<Checkbox
 isChecked={note.isSelected}
>
```

2. Add the client field into gql

```
query  GetAllNotes($categoryId: String, $offset: Int, $limit: Int) {
	notes(categoryId: $categoryId, offset: $offset, limit: $limit) {
	id
	content
	isSelected @client
		category {
			id
			label
		}
	}
}
```

3. Add into typePolicies in Cache:

```
cache: new  InMemoryCache({
	typePolicies: {
		Note: {
			fields: {
				isSelected: {
					read: () => {
						return  true;
					},
				},
			},
		},
	},
}),
```

4. Use the state stored in the central cache for other DOM elements:

- Query it:

```
query  GetNote($id: String!) {
	note(id: $id) {
		id
		content
			isSelected @client
		}
	}
}
```

- Use it:

```
<UiEditNote
	isNoteSelected={data?.note.isSelected}
/>
```

# How to change cache on update

## Problem

- We want the `read` typePolicy in cache to change if DOM changes

## Solution

- Use `makeVar` utility

1. Create a function to change the variable we want to listen to:

```
let  selectedNoteIds  =  makeVar(["2"]);
export function setNoteSelection(noteId, isSelected)  {
	if(isSelected) {
		selectedNoteIds([...selectedNoteIds(),  noteId]);
	}  else  {
		selectedNoteIds(selectedNoteIds().filter(selectedNoteId  =>  selectedNoteId  !==  noteId));
	}
}
```

2. Add into the read policy. use `helpers`

```
typePolicies:  {
	Note:  {
		fields:  {
			isSelected:  {
				read:  (currentIsSelectedValue,  helpers)  =>  {
					const  currentNoteId  =  helpers.readField("id");
						return  selectedNoteIds().includes(currentNoteId);
					},
				},
			},
		},
	},
```

3. Call the first function in the DOM

```
<Checkbox
	onChange={(e)  =>  setNoteSelection(note.id,  e.target.checked)}
	isChecked={note.isSelected}
>
```

# How to query without making network calls

## Problem

- When a user has slow internet or is offline, they aren't able to read data from without making network calls.

## Solution

- Can use Apollo's `typePolicy` to edit the `__ref` property of a query

```
note:  {
	read:  (existingCachedValue,  helpers)  =>  {
		const  queriedNoteId  =  helpers.args.id;
		return  {
			__ref:  `Note:${queriedNoteId}`,
		};
	},
},
```

- Use `helper` to replace `__ref` as it's an Apollo private field to their library.

```
const  queriedNoteId  =  helpers.args.id;
return  helpers.toReference({
	id:  queriedNoteId,
	__typename:  "Note",
});
```

# How to use `pollInterval`

## Problem

- We don't know if a new category is added into the server if no event triggers a query from the UI

## Solution

- Use `PollInterval `

```
const {data} = useQuery(ALL_CATEGORIES_QUERY, {
	pollInterval: 1000
}
```

# How to update notes without using the `pollInterval`

## Problem

- Notes comes with long texts, keep polling them will slow down the App

## Solution

- Use `subscription` query

1. Construct the query

```
const  {  data:  newNoteData  }  =  useSubscription(
gql`
	subscription  newSharedNote($categoryId:  String!) {
		newSharedNote(categoryId:  $categoryId) {
			id
			content
			category  {
				id
				label
			}
		}
	}
	`,
	{
		variables:  {
			categoryId:  category,
		},
	}
);
```

2. Setup the `websocketLink`

```
import {  WebSocketLink  } from "@apollo/client/link/ws";
const  wsLink  =  new  WebSocketLink({
	uri:  "ws://localhost:4000/graphql",
});
```

3. Conditionally render to use `websocketLink` or `httpLink`

```
import {  getMainDefinition  } from "@apollo/client/utilities";
const  protocolLink  =  split(
	({  query  })  =>  {
		const  definition  =  getMainDefinition(query);
			return  definition.operation  ===  "subscription";
		}
		wsLink,
		httpLink
);
```

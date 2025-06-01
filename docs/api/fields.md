<a name="beanie.odm.fields"></a>
# beanie.odm.fields

<a name="beanie.odm.fields.Indexed"></a>
#### Indexed

```python
Indexed(typ, index_type=ASCENDING, **kwargs)
```

Returns a subclass of `typ` with an extra attribute `_indexed` as a tuple:
- Index 0: `index_type` such as `pymongo.ASCENDING`
- Index 1: `kwargs` passed to `IndexModel`
When instantiated the type of the result will actually be `typ`.

<a name="beanie.odm.fields.PydanticObjectId"></a>
## PydanticObjectId Objects

```python
class PydanticObjectId(ObjectId)
```

Object Id field. Compatible with Pydantic.

<a name="beanie.odm.fields.Link"></a>
## Link Objects

```python
class Link(Generic[T])
```

Link field for referencing other documents. Links now store the `ObjectId` (or the specific ID type of the linked document `T`) directly in the database, providing better MongoDB compatibility and simplified querying.

<a name="beanie.odm.fields.Link.__init__"></a>
#### \_\_init\_\_

```python
 | __init__(obj_id: PydanticObjectId, document_class: Type[T])
```

Initialize a Link with an ObjectId and document class.

**Arguments**:

- `obj_id`: The ObjectId of the linked document
- `document_class`: The class of the document being linked to

<a name="beanie.odm.fields.Link.to_ref"></a>
#### to\_ref

```python
 | to_ref() -> PydanticObjectId
```

Returns the ObjectId of the linked document directly.

**Returns**:

PydanticObjectId: The ObjectId of the linked document

<a name="beanie.odm.fields.Link.to_dict"></a>
#### to\_dict

```python
 | to_dict() -> str
```

Returns the string representation of the ObjectId for JSON serialization.

**Returns**:

str: String representation of the ObjectId

**Notes**:

- Link fields are stored as ObjectIds directly in MongoDB, not as DBRef objects
- During validation, Link can accept either an ObjectId or a full document object
- For backward compatibility, DBRef objects may be accepted during validation when reading old data
- In JSON representation, Links appear as ObjectId strings
- Querying linked documents by ID uses direct field comparison: `MyDoc.link_field == object_id`

<a name="beanie.odm.fields.ExpressionField"></a>
## ExpressionField Objects

```python
class ExpressionField(str)
```

<a name="beanie.odm.fields.ExpressionField.__getattr__"></a>
#### \_\_getattr\_\_

```python
 | __getattr__(item)
```

Get sub field

**Arguments**:

- `item`: name of the subfield

**Returns**:

ExpressionField


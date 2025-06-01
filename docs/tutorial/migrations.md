## Attention!


## Create

To create a new migration, run:

```shell
beanie new-migration -n migration_name -p relative/path/to/migrations/directory/
```

It will create a file named `*_migration_name.py` in the directory `relative/path/to/migrations/directory/`

Migration file contains two classes: `Forward` and `Backward`. 
Each one contains instructions to roll migration respectively forward and backward.

## Run

**Attention**: By default, migrations use transactions. This approach only works with  **MongoDB replica sets**. If you prefer to run migrations without transactions, pass the `--no-use-transaction` flag to the `migrate` command. However, be aware that this approach is risky, as there is no way to roll back migrations without transactions.

To roll one forward migration, run:

```shell
beanie migrate -uri 'mongodb+srv://user:pass@host' -db db -p relative/path/to/migrations/directory/ --distance 1
```

To roll all forward migrations, run:

```shell
beanie migrate -uri 'mongodb://user:pass@host' -db db -p relative/path/to/migrations/directory/
```

To roll one backward migration, run:

```shell
beanie migrate -uri 'mongodb+srv://user:pass@host' -db db -p relative/path/to/migrations/directory/ --distance 1 --backward
```

To roll all backward migrations, run:

```shell
beanie migrate -uri 'mongodb+srv://user:pass@host' -db db -p relative/path/to/migrations/directory/ --backward
```

To show the help message with all the parameters and descriptions, run:

```shell
beanie migrate --help
```

## Migration types

Migration class contains instructions - decorated async functions. There are two types of instructions:

- Iterative migration - instruction that iterates over all the documents of the input_document collection and updates it. Most convenient to use, should be used in 99% cases.
- Free fall migrations - instruction where user can write any logic. Most flexible, but verbose.

### Iterative migrations

To mark a function as iterative migration, `@iterative_migration()` decorator must be used. 
The function itself must accept typed `input_document` and `output_document` arguments. Like here:

```python
@iterative_migration()


async def name_to_title(
        self, input_document: OldNote, output_document: Note
):
```

#### A simple example of field name changing

There are the next models:

```python
class Tag(BaseModel):
    color: str
    name: str


class OldNote(Document):
    name: str
    tag: Tag

    class Settings:
        name = "notes"


class Note(Document):
    title: str
    tag: Tag

    class Settings:
        name = "notes"

```

To migrate from `OldNote` to `Note`, file `name` has to be renamed to `title`.

Forward migration:

```python
class Forward:
    @iterative_migration()
    async def name_to_title(
            self, input_document: OldNote, output_document: Note
    ):
        output_document.title = input_document.name

```

Backward migration:

```python
class Backward:
    @iterative_migration()
    async def title_to_name(
            self, input_document: Note, output_document: OldNote
    ):
        output_document.name = input_document.title
```

And a little more complex example:

```python
from pydantic.main import BaseModel

from beanie import Document, iterative_migration


class OldTag(BaseModel):
    color: str
    name: str


class Tag(BaseModel):
    color: str
    title: str


class OldNote(Document):
    title: str
    tag: OldTag

    class Settings:
        name = "notes"


class Note(Document):
    title: str
    tag: Tag

    class Settings:
        name = "notes"


class Forward:
    @iterative_migration()
    async def change_color(
            self, input_document: OldNote, output_document: Note
    ):
        output_document.tag.title = input_document.tag.name


class Backward:
    @iterative_migration()
    async def change_title(
            self, input_document: Note, output_document: OldNote
    ):
        output_document.tag.name = input_document.tag.title
```
All the examples of migrations can be found by [link](https://github.com/roman-right/beanie/tree/main/tests/migrations/migrations_for_test)

### Free fall migrations

It is a much more flexible migration type, which allows the implementation of any migration logic. 
But at the same time, it is more verbose.

To mark function as a free fall migration, 
`@free_fall_migration()` decorator with the list of Document classes must be used. 
Function itself accepts `session` as an argument. 
It is used in order to roll back the migration in case something has gone wrong. 
To be able to roll back, please pass session to the Documents methods. Like here:

```python
@free_fall_migration(document_models=[OldNote, Note])
async def name_to_title(self, session):
    async for old_note in OldNote.find_all():
        new_note = Note(
            id=old_note.id, title=old_note.name, tag=old_note.tag
        )
        await new_note.replace(session=session)
```

#### The same example as for the iterative migration, but with free fall migration type

```python
from pydantic.main import BaseModel

from beanie import Document, free_fall_migration


class Tag(BaseModel):
    color: str
    name: str


class OldNote(Document):
    name: str
    tag: Tag

    class Settings:
        name = "notes"


class Note(Document):
    title: str
    tag: Tag

    class Settings:
        name = "notes"


class Forward:
    @free_fall_migration(document_models=[OldNote, Note])
    async def name_to_title(self, session):
        async for old_note in OldNote.find_all():
            new_note = Note(
                id=old_note.id, title=old_note.name, tag=old_note.tag
            )
            await new_note.replace(session=session)


class Backward:
    @free_fall_migration(document_models=[OldNote, Note])
    async def title_to_name(self, session):
        async for old_note in Note.find_all():
            new_note = OldNote(
                id=old_note.id, name=old_note.title, tag=old_note.tag
            )
            await new_note.replace(session=session)

```
All the examples of migrations can be found by [link](https://github.com/roman-right/beanie/tree/main/tests/migrations/migrations_for_test)

### Migrating Link Fields from DBRef to ObjectId

**Important**: This is a significant breaking change for users with existing data. Previously, `Link` fields were stored as `DBRef` objects, but now they are stored as direct `ObjectId`s.

If you have existing data with `Link` fields stored as `DBRef`s, you will need to migrate them. Here's how to do it:

#### Iterative Migration Example

```python
from beanie import Document, iterative_migration, Link
from typing import Any
from bson import DBRef

# Temporary model to read old DBRef data
class OldHouseWithDBRef(Document):
    name: str
    door: Any  # Can parse both DBRef and new Link format

    class Settings:
        name = "houses"  # same collection

class NewHouseWithObjectId(Document):
    name: str
    door: Link[Door]  # New Link type storing ObjectId

    class Settings:
        name = "houses"

class Forward:
    @iterative_migration()
    async def migrate_door_link(
            self, input_document: OldHouseWithDBRef, output_document: NewHouseWithObjectId
    ):
        # Handle single link field
        if isinstance(input_document.door, DBRef):
            output_document.door = input_document.door.id
        elif hasattr(input_document.door, 'obj_id'):  # if it parsed into new Link already
            output_document.door = input_document.door.obj_id
        
        # For list of links, handle each item
        # if hasattr(input_document, 'windows') and input_document.windows:
        #     output_document.windows = []
        #     for window_link in input_document.windows:
        #         if isinstance(window_link, DBRef):
        #             output_document.windows.append(window_link.id)
        #         elif hasattr(window_link, 'obj_id'):
        #             output_document.windows.append(window_link.obj_id)

class Backward:
    @iterative_migration()
    async def migrate_door_link_back(
            self, input_document: NewHouseWithObjectId, output_document: OldHouseWithDBRef
    ):
        # Convert ObjectId back to DBRef for backward compatibility
        if input_document.door:
            output_document.door = DBRef("doors", input_document.door)
```

#### Important Notes

- **Backup your data** before running such migrations
- Test the migration on a copy of your data first
- The new query syntax for linked documents by ID is: `MyDoc.link_field == object_id` instead of `MyDoc.link_field.id == object_id`
- Beanie's updated `Link` validation can handle reading old `DBRef` data during the migration process

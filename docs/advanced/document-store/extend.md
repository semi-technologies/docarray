# Add New Document Store

DocumentArray can be easily extended to support different Document Store backend. A Document Store can be a SQL/NoSQL/vector database, or even an in-memory data structure. The motivation of onboarding a new Document Store backend is often:
- having persistence that better fits to the use case;
- pulling from an existing data source;
- supporting advanced query languages, e.g. nearest-neighbor retrieval.

After the extension, users can enjoy convenient and powerful DocumentArray API on top of your document store. To end users, it promises the same user experience just like using a regular DocumentArray, no extra learning is required.

This chapter gives you a walkthrough on how to add a new DocumentArray. To be specific, in this chapter we are extending DocumentArray to support a new backend `mydocstore`. The final result would be enabling the following usage:

```python
from docarray import DocumentArray

da = DocumentArray(storage='mydocstore')
```

Let's get started!

## Step 1: create the folder

Go to `docarray/array/storage` folder, create a sub-folder for your document store. Let's call it `mydocstore`. You will need to create four empty files in that folder:

```{code-block} 
---
emphasize-lines: 8-13
---
README.md
docarray
    |
    |--- array
            |
            |--- storage
                    |
                    |--- mydocstore
                            |
                            |--- __init__.py
                            |--- getsetdel.py
                            |--- seqlike.py
                            |--- backend.py
```

These four files consist of necessary interface for making the extension work on DocumentArray.

## Step 2: implement `getsetdel.py` 

Your `getsetdel.py` should look like the following:

```python
from docarray.array.storage.base.getsetdel import BaseGetSetDelMixin
from docarray import Document

class GetSetDelMixin(BaseGetSetDelMixin):

    def _get_doc_by_offset(self, offset: int) -> 'Document':
        # to be implemented
        
    def _get_doc_by_id(self, _id: str) -> 'Document':
        # to be implemented

    def _del_doc_by_offset(self, offset: int):
        # to be implemented

    def _del_doc_by_id(self, _id: str):
        # to be implemented

    def _set_doc_by_offset(self, offset: int, value: 'Document'):
        # to be implemented

    def _set_doc_by_id(self, _id: str, value: 'Document'):
        # to be implemented
```

You will need to implement the above six functions, which correspond to the logics of get/set/delete items via a string `.id` or an integer offset. They are essential to ensure DocumentArray works.

```{tip}
Let's call the above six functions as **six essentials**.

If you aim for high performance, it is recommened to implement other methods *without* leveraging your six essentials. They are: `_get_docs_by_slice`, `_get_docs_by_offsets`, `_get_docs_by_ids`, `_del_docs_by_slice`, `_del_docs_by_mask`, `_del_all_docs`, `_set_docs_by_slice`, `_set_doc_value_pairs`, `_set_doc_value_pairs_nested`, `_set_doc_attr_by_offset`. One can get their full signatures from {class}`~docarray.array.storage.base.getsetdel.BaseGetSetDelMixin`. These functions define more fine-grained get/set/delete logics that are frequently used in DocumentArray. 

Implementing them is fully optional, and you can only implement some of them not all of them. If you are not implementing them, those methods will use a generic-but-slow version that based on your six essentials.
```

```{seealso}
As a reference, you can check out how we implement for SQLite, check out {class}`~docarray.array.storage.sqlite.getsetdel.GetSetDelMixin`.
```

## Step 3: implement `seqlike.py`

Your `seqlike.py` should look like the following:

```python
from typing import MutableSequence, Iterable, Iterator, Union
from docarray import Document

class SequenceLikeMixin(MutableSequence[Document]):
    
    def insert(self, index: int, value: 'Document'):
        ...

    def clear(self) -> None:
        ...

    def __eq__(self, other):
        ...
    
    def __len__(self):
        ...

    def __iter__(self) -> Iterator['Document']:
        ...

    def __contains__(self, x: Union[str, 'Document']):
        ...
    
    def __bool__(self):
        ...
    
    def __repr__(self):
        ...
    
    def __add__(self, other: Union['Document', Iterable['Document']]):
        ...

    def append(self, value: 'Document'):
        # optional, if you have better implementation than `insert`

    def extend(self, values: Iterable['Document']) -> None:
        # optional, if you have better implementation than `insert` one by one
```

Most of the interfaces come from Python standard [MutableSequence](https://docs.python.org/3/library/collections.abc.html#collections.abc.MutableSequence).

```{seealso}
As a reference, to see how we implement for SQLite, check out {class}`~docarray.array.storage.sqlite.seqlike.SequenceLikeMixin`.
```

## Step 4: implement `backend.py`

Your `backend.py` should look like the following:

```python
from typing import (
    Optional,
    TYPE_CHECKING,
    Union,
    Dict
)
from dataclasses import dataclass

from docarray.array.storage.base.backend import BaseBackendMixin

if TYPE_CHECKING:
    from docarray.types import (
        DocumentArraySourceType,
    )
    
@dataclass
class MyDocStoreConfig:
    config1: str
    config2: str
    config3: Dict
    ...

class BackendMixin(BaseBackendMixin):
    
    def _init_storage(
        self, 
            _docs: Optional['DocumentArraySourceType'] = None, 
            config: Optional[Union[MyDocStoreConfig, Dict]] = None,
            **kwargs
    ):
        ...
```

`_init_storage` is a very important function to be called during the DocumentArray construction. You will need to handle different construction & copy behaviors in this function.

`MyDocStoreConfig` is a dataclass for containing the configs. You can expose arguments of your document store to this data class and allow users to customize them. In `init_storage` function, you need to parse `config` either from `MyDocStoreConfig` object or a `Dict`.

```{seealso}
As a reference, you can check out how we implement for SQLite, check out {class}`~docarray.array.storage.sqlite.backend.BackendMixin`.
```


## Step 4: summarize everything in `__init__.py`.

Your `__init__.py` should look like the following:

```python
from abc import ABC

from .backend import BackendMixin, MyDocStoreConfig
from .getsetdel import GetSetDelMixin
from .seqlike import SequenceLikeMixin

__all__ = ['StorageMixins', 'MyDocStoreConfig']


class StorageMixins(BackendMixin, GetSetDelMixin, SequenceLikeMixin, ABC):
    ...

```

Just copy-paste it will do the work.

## Step 5: subclass from `DocumentArray`

Create a file `mydocstore.py` under `docarray/array/`

```{code-block}
---
emphasize-lines: 6
---
README.md
docarray
    |
    |--- array
            |
            |--- mydocstore.py
            |--- storage
                    |
                    |--- mydocstore
                            |
                            |--- __init__.py
                            |--- getsetdel.py
                            |--- seqlike.py
                            |--- backend.py
```

The file content should look like the following:

```python
from .document import DocumentArray

from .storage.mydocstore import StorageMixins, MyDocStoreConfig

__all__ = ['MyDocStoreConfig', 'DocumentArrayMyDocStore']


class DocumentArrayMyDocStore(StorageMixins, DocumentArray):
    def __new__(cls, *args, **kwargs):
        return super().__new__(cls)

```


## Step 6: add entrypoint to `DocumentArray`

We are almost there! Now we need to add the entrypoint to `DocumentArray` constructor to allow user to use the `mydocstore` backend as follows:

```python
from docarray import DocumentArray

da = DocumentArray(storage='mydocstore')
```

Go to `docarray/array/document.py` and add `mydocstore` there:

```{code-block} python
---
emphasize-lines: 7-10
---
class DocumentArray(AllMixins, BaseDocumentArray):
    
    ...
    
    def __new__(cls, *args, storage: str = 'memory', **kwargs) -> 'DocumentArrayLike':
        if cls is DocumentArray:
            if storage == 'mydocstore':
                from .mydocstore import DocumentArrayMyDocStore

                instance = super().__new__(DocumentArrayMyDocStore)
            elif storage == 'memory':
                from .memory import DocumentArrayInMemory
                ...  
```

Done! Now you should be able to use it like `DocumentArrayMyDocStore`!

## Contribute: add tests and type-hint

Welcome to contribute your extension back to DocArray. You will need to include `DocumentArrayMyDocStore` in at least the following tests:

```text
tests/unit/array/test_advance_indexing.py
tests/unit/array/test_sequence.py
tests/unit/array/test_construct.py
```

Please also add `@overload` type hint to `docarray/array/document.py`.

```python

class DocumentArray(AllMixins, BaseDocumentArray):
    ...
    
    @overload
    def __new__(
        cls,
        _docs: Optional['DocumentArraySourceType'] = None,
        storage: str = 'mydocstore',
        config: Optional[Union['MyDocStoreConfig', Dict]] = None,
    ) -> 'DocumentArrayMyDocStore':
        """Create a MyDocStore-powered DocumentArray object."""
        ...
```

Now you are ready to commit the contribution and open a pull request. 
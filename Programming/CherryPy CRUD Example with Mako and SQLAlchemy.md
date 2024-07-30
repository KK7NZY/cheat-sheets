#python #web #programming

This basic CRUD application utilizes the [CherryPy](https://cherrypy.dev/) framework, the Mako template engine, and [SQLAlchemy ORM](https://www.sqlalchemy.org/). While the code is several years old and incomplete, it remains a meaningful part of my early programming journey. These were the first libraries I encountered, and I found them a lot of fun to learn and use compared to the PHP tools available at the time. They made learning Python an enjoyable experience and contributed to my enthusiasm for the language. I’ve kept this code as a reference for any future projects where I could build a lightweight API. Unfortunately, I doubt that time will ever come, as there are now many newer libraries like FastAPI, which I also enjoy and, in my opinion, are better.

__Project Structure__

```shell
.
├── Pipfile
├── Pipfile.lock
├── lib
│   ├── __init__.py
│   └── cp_mako.py
├── main.py
├── notes.db
├── static
│   └── style.css
└── templates
    ├── form.html
    ├── index.html
    ├── layout.html
    └── note.html
```

__Mako Plugin__

I found this [snippet](https://gist.github.com/eneldoserrata/6058794) on GitHub when searching for CherryPy plugins. I made some small tweaks but nothing major.

```python
# lib/cp_mako.py
import cherrypy
from cherrypy.process.plugins import SimplePlugin
from mako import exceptions
from mako.lookup import TemplateLookup

# https://gist.github.com/eneldoserrata/6058794
class MakoPlugin(SimplePlugin):

    def __init__(self, bus, directories=None, module_directory=None, collection_size=50, encoding="utf-8"):
        SimplePlugin.__init__(self, bus)
        self.directories = directories
        self.module_directory = module_directory
        self.encoding = encoding
        self.collection_size = collection_size
        self.lookup = None
        self.subscribe()

    def start(self):
        self.lookup = TemplateLookup(
            directories=self.directories,
            module_directory=self.module_directory,
            input_encoding=self.encoding,
            output_encoding=self.encoding,
            collection_size=self.collection_size
        )
        self.bus.subscribe("mako.lookup", self.serve_template)

    def stop(self):
        self.bus.unsubscribe("mako.lookup", self.serve_template)
        self.lookup = None

    def serve_template(self, template):
        return self.lookup.get_template(template)


class MakoTool(cherrypy.Tool):

    def __init__(self):
        cherrypy.Tool.__init__(self, "before_finalize", self._render, priority=30)

    def _render(self, filename=None, debug=False):
        data = cherrypy.response.body or {}
        template = None if filename is None else cherrypy.engine.publish("mako.lookup", filename).pop()

        if template and isinstance(data, dict):
            if debug:
                try:
                    cherrypy.response.body = template.render(**data)
                except Exception:
                    cherrypy.response.body = exceptions.html_error_template().render()
            else:
                cherrypy.response.body = template.render(**data)

```


__Main Module__

The main module of this application sets up the CherryPy server and integrates the various components of the CRUD application. It configures the database, mounts the CherryPy handlers, and initializes the Mako and SQLAlchemy plugins. The following code outlines the core structure and functionality of the application, including how it manages database operations and handles HTTP requests.

```python
import os.path
from datetime import datetime

import cherrypy
from cp_sqlalchemy import SQLAlchemyTool, SQLAlchemyPlugin
from sqlalchemy import Column, DateTime, ForeignKey, Integer, String, Text
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship, backref
from markdown import markdown

from lib.cp_mako import MakoPlugin, MakoTool


BaseEntity = declarative_base()


class Tag(BaseEntity):
    __tablename__ = "tag"

    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False, unique=True)


class Category(BaseEntity):
    __tablename__ = "category"

    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False, unique=True)


class Note(BaseEntity):
    __tablename__ = "note"

    id = Column(Integer, primary_key=True)
    title = Column(String, nullable=False)
    _markdown = Column("markdown", Text)
    created_at = Column(DateTime, nullable=False, default=datetime.utcnow)
    updated_at = Column(DateTime, onupdate=datetime.utcnow)
    # https://docs.sqlalchemy.org/en/14/orm/basic_relationships.html#many-to-many
    tag_id = Column(Integer, ForeignKey("tag.id"))
    tags = relationship(Tag, backref=backref("notes"))
    category_id = Column(Integer, ForeignKey('category.id'))
    categories = relationship(Category, backref=backref("notes"))

    # https://docs.sqlalchemy.org/en/14/orm/mapped_attributes.html#using-descriptors-and-hybrids
    @property
    def markdown(self):
        return markdown(self._markdown)

    @markdown.setter
    def markdown(self, md):
        self._markdown = md


# register tools
# https://docs.cherrypy.org/en/3.3.0/progguide/extending/customtools.html
cherrypy.tools.db = SQLAlchemyTool()
cherrypy.tools.render = MakoTool()


class Root:

    @property
    def db(self):
        return cherrypy.request.db

    # https://docs.cherrypy.org/en/latest/pkg/cherrypy._cpconfig.html?highlight=%40cherrypy.config#declaration
    @cherrypy.expose
    @cherrypy.tools.render(filename="index.html")
    def index(self):
        return {"notes": self.db.query(Note).all()}

    @cherrypy.expose
    @cherrypy.tools.render(filename="note.html")
    def read(self, pk):
        return {"note": self.db.query(Note).get(pk)}

    @cherrypy.expose
    @cherrypy.tools.render(filename="form.html")
    def edit(self, pk=None):
        if pk is not None:
            note = self.db.query(Note).get(pk)
            resp = {"note": note, "action": "update"}
        else:
            resp = {"action": "create"}
        return resp

    @cherrypy.expose
    def create(self, title, md):
        method = cherrypy.request.method

        if method == "POST":
            note = Note(markdown=md, title=title)

            self.db.add(note)
            self.db.commit()

        raise cherrypy.HTTPRedirect('/')

    @cherrypy.expose
    def update(self, pk, title, md):
        return

    @cherrypy.expose
    def delete(self, pk):
        note = self.db.query(Note).get(pk)
        self.db.delete(note)
        self.db.commit()

        raise cherrypy.HTTPRedirect('/')


def main():
    base_path = os.path.dirname(__file__)

    # configure and mount handlers
    config = {
        "/": {
            "tools.db.on": True,
            "tools.render.on": True,
            "tools.encode.on": False
        },
        "/static": {
            "tools.staticdir.on": True,
            "tools.staticdir.dir": os.path.join(base_path, "static"),
            "tools.gzip.on": True,
            # https://docs.cherrypy.org/en/latest/pkg/cherrypy.lib.encoding.html?highlight=tools.gzip.mime_types#cherrypy.lib.encoding.gzip
            "tools.gzip.mime_types": ["text/*", "application/*"]
        }
    }
    cherrypy.tree.mount(Root(), '/', config=config)

    # setup sql database
    db_filename = os.path.join(base_path, "notes.db")

    if not os.path.exists(db_filename):
        with open(db_filename, 'a'):
            pass

    # setup plugins
    mako_plugin = MakoPlugin(cherrypy.engine, os.path.join(base_path, "templates"))
    mako_plugin.subscribe()

    # https://docs.sqlalchemy.org/en/14/tutorial/engine.html
    sqlalchemy_plugin = SQLAlchemyPlugin(cherrypy.engine, BaseEntity, f"sqlite+pysqlite:///{db_filename}", echo=True)
    sqlalchemy_plugin.subscribe()
    sqlalchemy_plugin.create()

    # start server
    cherrypy.engine.start()
    cherrypy.engine.block()


if __name__ == '__main__':
    main()
```
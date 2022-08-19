This article is about ipychess, an interactive chessboard component for the Jupyter Notebook that I've recently made with Python and JavaScript. The component is based on the famous Chessground library and relies heavily on Jupyter's “comms” feature. More on that below

# Table of contents
- [Table of contents](#table-of-contents)
- [The problematic: interacting with remote chess engines and data is tricky](#the-problematic-interacting-with-remote-chess-engines-and-data-is-tricky)
- [Interactive widgets in Jupyter/JupyterLab](#interactive-widgets-in-jupyterjupyterlab)
- [My solution: bringing the power of Lichess into the notebook](#my-solution-bringing-the-power-of-lichess-into-the-notebook)
  - [Custom widget template](#custom-widget-template)
  - [The "Board" component](#the-board-component)
    - [Python (Kernel) side](#python-kernel-side)
    - [Javascript(Typescript) side](#javascripttypescript-side)
- [Where do we go from here ?](#where-do-we-go-from-here-)
# The problematic: interacting with remote chess engines and data is tricky
As a long-time chess enthusiast I've always been interested in chess GUIs and engines. My interest for the subject led me to take a look at the Leela Chess Zero (lc0) project. 

Since most contributors didn't have powerful GPUs, it was (and is still) common practise to train the engine by using cloud services such as Kaggle and Google Colaboratory. Troubleshooting Leela or investigating specific weaknesses of the engine (fortresses, unbalanced endgames, long tactical sequences,...) after a TCEC match or major config change was harder to conduct since there wasn't yet a way to quickly efficiently interact with chess data in Jupyter notebooks. As a consequence, that part of the engine 

Although it has significantly grown its reach over the years, the Jupyter ecosytem's core tenet is about making interactive computing as accessible as possible. This project was no different in that regard.
 

# Interactive widgets in Jupyter/JupyterLab
Interactive widgets such as sliders have been a part of the Jupyter project since its inception as a spin-off or IPython (Interactive Python). That said, the ideas behind ipywidgets as they appear now, are quite recent in comparison and have been motivated by the notebook's increasing use in a cloud context, were network latency and limited throughput became major concerns for the scientific computing community.

 Without entering into too much detail, the essence of ipywidgets is that responsibilities (data processing and rendering) should be better shared between a notebook's kernel (back-end) and its client. Since, most clients are powerful enough to render, let's say a chart, this shouldn't be done by the Python kernel. Rendering a chart in the kernel, then sending an image through the network to be displayed in a notebook wastes both precious server CPU time and network bandwidth. Ipywidgets solve this problem by allowing the client to run some Javascript code to render a widget based on relatively lightweight information shared with the kernel.
 
 The following image summarizes what the Comms protocol that underlies ipywidgets is all about:
 ![ipywidgets comms](/img/transport.svg "ipywidgets comms")

Once these concerns are properly separated, we gain the ability to even decouple the front-end state from the kernel, perform asynchronous updates, duplicate widgets based on a common state without wasting expensive server and network resources and finally interact with the widget through the DOM.
# My solution: bringing the power of Lichess into the notebook

Lichess (lichess.org) is an open source chess website that routinely handles more than 30,000 concurrent games, among which several prized tournaments... some of which can be endorsed by the international chess federation.

 ![ipywidgets comms](/img/Lichess_study.png "ipywidgets comms")

Lichess's clean and performant design comes, in part, from the minimalist library it uses to render chess boards: Chessground.js. I'll use this library as a basis for the front-end of my solution. The rest will come from ipywidgets proper.


## Custom widget template

Ipywidgets can be easily customized to wrap any javascript library by creating a [cookiecutter](https://github.com/cookiecutter/cookiecutter) template for our custom
Jupyter widget project.


With jupyter's own **widget-ts-cookiecutter**, we'll then create a custom Jupyter interactive
widget project with sensible defaults. widget-ts-cookiecutter helps custom widget
authors get started with best practices for the packaging and distribution
of a custom Jupyter interactive widget library.

Here are the steps I followed, as described in the official repo:

- Install [cookiecutter](https://github.com/audreyr/cookiecutter):

    ```bash
    pip install cookiecutter
    ```

- After installing cookiecutter, we use widget-ts-cookiecutter:

    cookiecutter https://github.com/jupyter-widgets/widget-ts-cookiecutter.git

  As widget-ts-cookiecutter runs, you will be asked for basic information about
  your custom Jupyter widget project. You will be prompted for the following
  information:

  - `author_name`: your name or the name of your organization,
  - `author_email`: your project's contact email,
  - `github_project_name`: name of your custom Jupyter widget's GitHub repository,
  - `github_organization_name`: name of your custom Jupyter widget's GitHub user or organization,
  - `python_package_name`: name of the Python "back-end" package used in your custom widget.
  - `npm_package_name`: name for the npm "front-end" package holding the JavaScript
    implementation used in your custom widget.
  - `npm_package_version`: initial version of the npm package.
  - `project_short_description` : a short description for your project that will
    be used for both the "back-end" and "front-end" packages.


After renaming a few default files and updating references, I ended up with the basis of my own custom widget library:

![ipychess file structure](/img/files.png "Ipychess file structure")


I then created a GitHub repo for the project at https://github.com/Kalhina/ipychess
## The "Board" component
To create an interactive chess board component, all we have to do is

### Python (Kernel) side

First we create custom traitlet types (types.py) for the board component:

```python
from traitlets import HasTraits, Enum, Bool, Dict, List
import string
from itertools import product

colors = ['white', 'black'] 
files = string.ascii_lowercase[:8]
ranks = string.digits[1:9]


class Color(Enum):
    def __init__(self):
        self.default_value = 'white'
        self.values = colors


class Role(Enum):
    def __init__(self):
        self.values = ['king', 'queen', 'rook', 'bishop', 'knight', 'pawn']


class Key(Enum):
    def __init__(self):
        self.allow_none = True
        self.default_value = None
        self.values = ["a0"] + [f'{fl}{rk}' for fl, rk in product(files, ranks)]


```

Then, in board.py, we extend the DOMWidget class with attributes inspired from Chessground

```python
#!/usr/bin/env python
# coding: utf-8

# Copyright (c) uribalb.
# Distributed under the terms of the Modified BSD License.

"""
TODO: Add module docstring
"""

from ipywidgets import DOMWidget
from traitlets import Unicode, Union, List, Dict, Bool, default

from ._frontend import module_name, module_version
from .types import Color, Key


class Board(DOMWidget):
    """TODO: Add docstring here
    """
    _model_name = Unicode('BoardModel').tag(sync=True)
    _model_module = Unicode(module_name).tag(sync=True)
    _model_module_version = Unicode(module_version).tag(sync=True)
    _view_name = Unicode('BoardView').tag(sync=True)
    _view_module = Unicode(module_name).tag(sync=True)
    _view_module_version = Unicode(module_version).tag(sync=True)

    size = Unicode('400px').tag(sync=True)

    # Chessground options
    fen = Unicode(
        'rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1').tag(sync=True, o=True)
    orientation = Color().tag(sync=True, o=True)
    turn_color = Color().tag(sync=True, o=True)
    check = Union([Color(), Bool(), Key()], allow_none=True, default_value=False).tag(
        sync=True, o=True)
    last_move = List(trait=Key()).tag(sync=True, o=True)
    selected = Key().tag(sync=True, o=True)
    coordinates = Bool(default_value=True).tag(sync=True, o=True)
    auto_castle = Bool(default_value=True).tag(sync=True, o=True)
    view_only = Bool(default_value=False).tag(sync=True, o=True)
    disable_context_menu = Bool(default_value=False).tag(sync=True, o=True)
    resizable = Bool(default_value=True).tag(sync=True, o=True)
    add_piece_z_index = Bool(default_value=False).tag(sync=True, o=True)
    highlight = Dict().tag(sync=True, o=True)
    animation = Dict().tag(sync=True, o=True)
    movable = Dict().tag(sync=True, o=True)
    premovable = Dict().tag(sync=True, o=True)
    predroppable = Dict().tag(sync=True, o=True)
    draggable = Dict().tag(sync=True, o=True)
    selectable = Dict().tag(sync=True, o=True)
    events = Dict().tag(sync=True, o=True)
    drawable = Dict().tag(sync=True, o=True)

    options = List(trait=Unicode()).tag(sync=True, o=False)

    @default('options')
    def _default_options(self):
        return [name for name in self.traits(o=True)]

```
The sync=True keyword argument tells the widget framework to handle synchronizing that value to the browser. Without sync=True, attributes of the widget won’t be synchronized with the front-end.


### Javascript(Typescript) side
We start by creating separate utility functions (utils.ts) mainly to handle strings, function debouncing (to avoid redrawing the board needlessly)
```js

export function camel_case(input: string): string {
  // Convert from foo_bar to fooBar
  return input.toLowerCase().replace(/_(.)/g, (match, group) => {
    return group.toUpperCase();
  });
}

export function snake_case(input: string): string {
  // Convert from fooBar to foo_bar
  return input.replace(/[A-Z]/g, letter => `_${letter.toLowerCase()}`)
}


export function debounce<F extends Function>(func: F, wait: number): F {
  let timeoutID: number;
  if (!Number.isInteger(wait)) {
    console.warn('Called debounce without a valid number');
    wait = 300;
  }
  // conversion through any necessary as it wont satisfy criteria otherwise
  return <any>function (this: any, ...args: any[]) {
    window.clearTimeout(timeoutID);
    const context = this;

    timeoutID = window.setTimeout(() => {
      func.apply(context, args);
    }, wait);
  };
}

export function emit_resize() {
  document.body.dispatchEvent(new Event('chessground.resize'));
}


```

Then (in Board.ts), we extend the DOMWidgetView to create the view for our Board object previously made in Python. The file is full of boilerpalte code but the most interesting part is the "initialize" and "render" methods (visit the repo for full access to the code):

```js
  initialize() {
    this.el.classList.add('brown', 'merida');
    this.el.style.width = this.model.get('size');
    this.el.style.height = this.model.get('size');

    this.board_container = document.createElement('div');

    this.el.appendChild(this.board_container);
    this.board_container.classList.add('jupyter-widgets');

    const scrollResizeHandler = debounce(emit_resize, 100);
    document.body.addEventListener('scroll', scrollResizeHandler, true);
    window.addEventListener('resize', scrollResizeHandler, true);
  }
  render() {
    super.render();
    this.model.on_some_change(
      Object.keys(this.get_options()),
      this.options_changed,
      this
    );
    requestAnimationFrame(this.render_chessground.bind(this));

  }

```

# Where do we go from here ?

A few ideas for improvements:

- Make a proper doc for this library. This article is only a tiny step in that direction

- "Advertise" the library in relevant chess programming communities to attract more experienced contributors

- Make an alternative API for the Board widget that allows the user to directly pass a json-like dict. This would more closely mimick Chessground's TS API and reduce the learning curve. We still have to figure out a way to properly serialize function arguments

- Add more widgets to the library that would ultimately enable us to create a full chess GUI within a jupyter notebook. We might need to use another widget library like "Bqplot" as a depency to achieve this without bloating ipychess itself

- Create more generic Board and add specialized components to handle other chess variants (Crazyhouse, KoTH, Racing Kings,...)

- Simplify the installation process (publish on PyPi)


Thanks for reading ! Feel free to suggest any improvement you'd like or contribute to the code !




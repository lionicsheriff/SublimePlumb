# AcmePlumbing
> Make your text clickable

+ [Configuration](#configuration)
+ [Extending](#extending)
+ [FAQ](#faq)
  + [Where is my context menu](#context-menu)

# <a name="what"></a> What

+ Right click on https://www.google.com/search?q=Acme+Editor and a google search is opened in your browser.
+ Right click on Commands.py@prepare_command and Commands.py is opened at the definition for prepare_command.
+ Right click on pydoc(re) to see help on python regular expressions.
+ Right click on shutdown and your computer turns off (causing you to wonder why you set up that last one).

# <a name="why"></a> Why

I played with Acme Editor and found the way it considered text to be part of the UI fun. However I wanted to play with it in an environment that I was more comfortable in (and runs nicely on Windows). Besides, I *really* wanted to be able to link files together as an adhoc wiki.

# <a name="how"></a> How

Select text with the right mouse button. The selected text is placed into a message as the data and is then passed to a set of commands (a rule). The commands are evaluated from the first to the last, and if one fails the rule stops processing and the next rule is tried.

If you only want to select a word, you can save effort by just right clicking in the middle of the word. This will cause AcmePlumbing to expand the selection along the word boundaries.


# <a name="configuration"></a> Configuration

> see AcmePlumbing (Linux).sublime-settings

Each rule is a list of commands to run. You can pass extra arguments to a command by wrapping it in a list.

```json
[
  "is_file",
  [ "pattern", "\.txt$"],
  "open_in_tab"
]
```

When a message is passed to this rule, the first commands checks that the message data refers to a file. If it is a file the next command is run, otherwise the rule exits. The second command then checks the message data against a regular expression. This command takes the regular expression as an argument. In this case it tests if the file is a .txt file. If that passes, the message is handed to the open_in_tab command, which opens the file referred to in the message data into a new tab.

NOTE: commands in the pipeline are free to modify the message (if the rule fails, the message is set back to the original for the next rule)

## Commands

### pattern

> see Commands.py@pattern

Runs the data against a regular expression specified in the second argument. The results are stored in the match\_data allowing the action pipeline to use segments of the data.

### is_file

> see Commands.py@is_file

is_file tests if the message data references a file. If it fails, it tries again as a relative path using the current working directory set in the message. If a file is found, the message data is set to the full path.

### is_dir

> see Commands.py@is_dir

is_dir tests if the message data references a directory. Like is_file, if it fails it tries again as a relative path using the current working directory set in the message. If a directory is found, the message data is set to the full path.

### list_dir

> see Commands.py@list_dir

list_dir assumes the message data is the path to a directory and lists it. Each item in the directory is expanded to its full path, and are separated by new lines. The message data is replaced with the list of items.

### extract_jump

> see Commands.py@extract_jump

This test is different as it will always pass. It's purpose is to remove jump locations form the data and store them somewhere separate (match\_data) so they don't interfere with subsequent tests. This is important as keeps stops is_file from having to be aware of how to jump to a location in a file, and it can focus on just testing if a file exists

> see Commands.py@jump for the syntax used to jump

### prepare_command

> see Commands.py@prepare_command

prepare_command replaces text in the data based off the results of the match pipeline. At its most basic $\_ is replaced with the contents of the message data (the text that you clicked on).

#### pattern

Results from the pattern test can be replaced by either referencing them by their group position (e.g. $1) or by the group name (e.g. $section)

### open\_in\_tab

> see Commands.py@open\_in\_tab

open\_in\_tab opens whatever is in message['data'] in a new tab. If a file exists with that path it will open that file. Otherwise it will assume that the data is a shell command. It will run the command and if there is output it will be placed in a new tab. An example of this is the rule to open man pages.

### display_data\_in_new\_tab

> see Commands.py@display_data_in_new_tab

display_data_in_new_tab creates a new tab and outputs the contents of the message data into it.

### jump

> see Commands.py@jump

jump uses the results from extract_jump and moves the cursor to a new location. It uses syntax similar to Go To Anything:

* __@__ jump to symbol
* __#__ jump to text
* __:__ jump to line

### open\_in\_external_process

> see Commands.py@open\_in\_external_process

open\_in\_external_process assumes that the message data is a command and runs it. No new tabs are opened. This is primarily used for rules like URLs where you want them to open in your browser, not your text editor.

### extern

> see Commands.py@extern

extern runs a command defined in an external module. The first argument is the module name, the second in the function, and the remainder are the arguments.

```json
["extern", "ExternalPlugin.Module", "custom_command", "arg1", "arg2", "arg3"]
```

### print_pipeline

> see Commands.py@print_pipeline

print_pipeline outputs the message and pipeline data at that point in the pipeline into the console. It is useful when debugging a pipeline.

# <a name="extending"></a> Extending

## Message

The structure of the message looks like

```json
{
  "data": "the selected text",
  "cwd": "the parent directory of the current file",
  "src": "the view id",
  "edit_token": "the edit token used for editing views"
}
```

## Creating new commands

You can add custom commands by creating them in AcmePlumbingCommands.py in your user directory. Each action is a function with the signature:

```python
def custom_command(message, arguments, pipeline_data):
  return True
```

The command must return a true value if it succeeds. Otherwise the rule will be considered failed and the next rule will
be run.

You can then reference them by the function name in your rule set:

```json
[ "custom_command" ]
```

The return value is placed into pipeline_data, a dictionary that contains the results of all the previous commands in the rule.

## Calling from another plugin

You can use SublimeAcmePlumbing.AcmePlumbing.add_rule (AcmePlumbing.py@add_rule) to inject a rule into the plumbing. This can be combined with the "extern" command to call a command defined in another module (e.g. another plugin). The rule is saved in the *user* settings to allow the user to tweak it and control its position in the plumbing. Additional rules have a key to allow the rule to be updated if it already exists, as such they should be unique.

> see AcmePlumbing.py@add_rule

```python
def add_rule(key, comment, rule)
```

The rule is saved in this format:

```javascript
// key
// comment
[
    "rule"
],
```

### Example

#### OtherPlugin.Plumbing.py
```python
import sublime
from AcmePlumbing import AcmePlumbing

def greet(message, args, match_data):
    window = sublime.active_window()
    tab = window.new_file()
    tab.set_scratch(True)
    edit_token = message['edit_token']
    tab.insert(edit_token, 0, "Hello. How's the weather?")
    return tab

def plugin_loaded():
    AcmePlumbing.add_rule("OtherPlugin.greet",
                          "Ask about the weather",
                          ["extern", "OtherPlugin.Plumbing", "greet"])
```

This plugin sets up the plumbing so that anything you right click on will open a new tab asking you about the weather

# <a name="faq"></a> FAQ

## <a name="context-menu"></a> Where is my context menu?

Since Acme Plumbing binds itself to the right mouse button, you can't access the right click menu normally. Don't panic: it is just a shift + right click away.

However, if you don't want Acme Plumbing on your right mouse button, you can move it to the middle mouse button by putting this into Default.sublime-mousemap in the Users package directory.

```json
[
  {
    "button": "button2", "count": 1, "modifiers": [],
    "press_command": "context_menu"
  },
  {
    "button": "button3", "count": 1, "modifiers": [],
    "command": "acme_plumbing_send",
    "press_command": "drag_select"
  },
]
```

# <a name="license"></a> License

MIT

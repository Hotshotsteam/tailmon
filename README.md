# tailmon
Triggers file system update events based on a file name log.

The primary purpose of this is to solve the lack of fs events in Virtualbox vboxsf.  The script is extremely simple.  Parsing arguments aside, it is simply:

```sh
tail -fn0 changed_files.log | \
while read change_file; do
  touch $change_file
done
```

The overall approach is more interesting than the script, due to it's simplicity:

- Host:  Append relative paths to changed files to a simple log file.
- Guest: Ingest the log file and ```touch``` the changed files.

This simple process triggers the fs events (such as inotify) that are needed by watchers running in the guest.

### Why not polling?
Anyone that has set up a file watcher in a Virtualbox VM with a standard vboxsf share will know that they don't work out of the box.  Most watchers provide a polling option, which polls all of the watched files for changes instead of listening to fs events.

Depending on the project complexity and number of files, the polling may saturate the CPU to the point that it's no longer useful. YMMV - I found polling with [lite-server](https://github.com/johnpapa/lite-server) on a simple [Angular2](https://github.com/angular/angular) project worked well, while polling an [Ionic2](https://github.com/driftyco/ionic) project with [watchify](https://github.com/substack/watchify) saturated the CPU and increased build times from 30ms to 5 minutes.


## Set up

### Virtualbox Guest

```sh
Usage:
  tailmon.sh [PROJECT_PATH LOG_FILE_PATH]

Options:
  PROJECT_PATH   Base path to trigger fs events [default: .]. 
  LOG_FILE_PATH  Path to log file with relative paths [default: ./.tailmon]
```

Run ```tailmon.sh``` to watch the log file.  It will be created if it doesn't exist.

### Host
The host needs to append the relative paths of the changed files to the log file specified above.  The log file should be shared with the guest; usually in the same share as the project.

There are many ways to achieve this.  I am using [atom](https://github.com/atom/atom) with the [savey-wavey](https://github.com/darthtrevino/savey-wavey) plugin.

#### atom
1. Install the [savey-wavey](https://github.com/darthtrevino/savey-wavey) plugin.
1. Create a ```tailmon.js``` to append the paths to the log file:

```js
var process = require('process');
var fs = require('fs');

if (process.argv.length < 4) {
  console.log('Usage: tailmon.js <path/to/changed/file> <path/to/tail/file>');
  throw "Not enough parameters"
}

var changed = process.argv[2];
var tail_file = process.argv[3];

// Remove if not Windows, or update for OSX
var changed = changed.replace(/\\/g, '/')  + "\n";

fs.appendFile(tail_file, changed);
```

1. Create a ```.on-save.json``` config for savey-wavey that calls the above script when a file is saved:

```js
{
  "commands": [
    {
      "watch": "**/app/**",
      "base": ".",
      "command": "node tailmon.js ${path} ${project}\\.tailmon"
    }
  ],
  "config": {
    "showSuccess": true,
    "autohideSuccess": false,
    "autohideSuccessTimeout": 1200
  }
}
```

See [savey-wavey](https://github.com/darthtrevino/savey-wavey) or more information.

That's it!  Run your watchers and get back to enjoying your live builds and reloads.

# Contributing
If you have set up steps for other host IDEs or watchers, please leave a comment on [this issue](https://github.com/Hotshotsteam/tailmon/issues/1).

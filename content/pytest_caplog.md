title: But It Looked Like a Nail
date: March 30, 2020
author: Andy Jarcho

# Hitting pytest's capsys with a hammer
### The problem
The code to be tested is from a script that keeps tabs (ahem) on the amount of caffeine in your body.

* If you call the script with no arguments, it gets your previous reading (level and time) from a .json file. Then it 
    * calculates your current caffeine level from that data based on a simple formula
    * writes the new level and time to stdout, and
    * overwrites the .json file with the new reading.

* To tell the script you just drank a cup of coffee with 240 mg of caffeine, you call it with the single argument 240.

* If you had a cup of coffee an hour ago, but forgot to 'tell' the script, just call it with the arguments 240 60.

Here's the method that overwrites the .json file, and maintains the log.
 
```python
class CaffeineMonitor:

    def write_file(self):
        self.iofile.seek(0)          # self.iofile is the .json file
        self.iofile.truncate(0)
        log_mesg = (f'level is {round(self.data_dict["level"], 1)}'  # level 
                    f'at {self.data_dict["time"]}')                  # and time
        if self.mg_to_add: 
            log_mesg = f'{self.mg_to_add} mg added: ' + log_mesg
            logging.info(log_mesg)   # just had caffeine: use log level INFO
        else:
            logging.debug(log_mesg)  # no new caffeine: log as DEBUG
        json.dump(self.data_dict, self.iofile)
```

### The process
I'd been struggling to test this method using pytest and pytest_mock (I'm learning about mocking and patching).
I got an object lesson in "when the only tool you have is a hammer, everything starts to look like a nail".

The main obstacle was capturing the log output for my tests.
I started out trying to use `mocker` to mock out the logging functionality.
Six hours later, I was *still* trying to use `mocker` to...

Then I ran across an [SO post](https://stackoverflow.com/questions/22657591/get-all-logging-output-with-mock) 
that introduced me to `caplog`. The relevant part of the post was the answer by @hoefling.

@hoefling's pytest example:
```python
# spam.py

import logging
logger=logging.getLogger(__name__)

def foo():
    logger.info('bar')


# tests.py

import logging
from spam import foo

def test_foo(caplog):
    foo()
    assert len(caplog.records) == 1
    record = next(iter(caplog.records))
    assert record.message == 'bar'
    assert record.levelno == logging.INFO
    assert record.module == 'spam'
    # etc
```

didn't work for me initially. `len(caplog.records)` turned out to be 0 here also, until I played with the code a bit.

### The solution

My root logger is set up as:
```python
# src/caffeine_monitor.py

    logging.basicConfig(filename=config[current_environment]['log_file'],
                        level=logging.DEBUG,
                        format='%(message)s')
```

The tests that work are: (`cm` and `test_files` are fixtures, set up in `conftest.py`, that return a `CaffeineMonitor` 
instance and a pair of file names, respectively)

```python
def test_write_file_add_mg(cm, test_files, caplog):
    with open(test_files[0], 'r+') as l_file_handle:
        cur_time = datetime.datetime.now().strftime('%Y-%m-%d_%H:%M')
        cm.data_dict = {'level': 140.0, 'time': cur_time}
        caplog.set_level('INFO')

        cm.mg_to_add = 140

        cm.write_file()

        assert f'140 mg added: level is 140.0 at {cur_time}' in caplog.text
        assert len(caplog.records) == 1


def test_write_file_add_no_mg(cm, test_files, caplog):
    with open(test_files[0], 'r+') as l_file_handle:
        cur_time = datetime.datetime.now().strftime('%Y-%m-%d_%H:%M')
        cm.data_dict = {'level': 140.0, 'time': cur_time}
        caplog.set_level('DEBUG')

        cm.write_file()

        assert f'level is 140.0 at {cur_time}' in caplog.text
        assert len(caplog.records) == 1
```

### The issue
Log levels (sigh). I'd assumed that, since the root logger level was set to `DEBUG`, any
message of that level or higher would be sent to `caplog`. It turns out that, like the root logger, `caplog`'s 
default level is `WARNING`. Hence caplog was not seeing the messages logged by `CaffeineMonitor.write_file()`.

### The lesson
Reading documentation is something of an art form. You often can't read all the docs word-for-word; parts of it
have to be skimmed. In this case I happened to skim, and did not absorb, the datum that would have saved me hours
of work. 






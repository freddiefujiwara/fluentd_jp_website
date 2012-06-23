.. _devel:

Writing plugins
========================

.. contents::
   :backlinks: none
   :local:

Installing custom plugins
------------------------------------

To install a plugin, put a ruby script to ``/etc/fluent/plugin`` directory.

Or you can create gem package that includes ``lib/fluent/plugin/<TYPE>_<NAME>.rb`` file. *TYPE* is ``in`` for input plugins, ``out`` for output plugins and ``buf`` for buffer plugins. It's like ``lib/fluent/plugin/out_mail.rb``. The packaged gem can be distributed and installed using RubyGems. See :ref:`search_plugin`.


Input plugins
------------------------------------

Extend **Fluent::Input** class and implement following methods.

.. code-block:: ruby

    class SomeInput < Fluent::Input
      # Register plugin first. NAME is the name of this plugin
      # which is used in the configuration file.
      Fluent::Plugin.register_input('NAME', self)

      # This method is called before starting.
      # 'conf' is a Hash that includes configuration parameters.
      # If the configuration is invalid, raise Fluent::ConfigError.
      def configure(conf)
        super
        @port = conf['port']
        ...
      end

      # This method is called when starting.
      # Open sockets or files and create a thread here.
      def start
        super
        ...
      end

      # This method is called when shutting down.
      # Shutdown the thread and close sockets or files here.
      def shutdown
        ...
      end
    end

To submit events, use ``Fluent::Engine.emit(tag, time, record)`` method, where ``tag`` is the String, ``time`` is the UNIX time integer and ``record`` is a Hash object.

.. code-block:: ruby

    tag = "myapp.access"
    time = Time.now.to_i
    record = {"message"=>"body"}
    Fluent::Engine.emit(tag, time, record)

RDoc of the Engine class is available from `Fluentd RDoc <http://fluentd.org/rdoc/Fluent/Engine.html>`_.


Buffered output plugins
------------------------------------

Extend **Fluent::BufferedOutput** class and implement following methods.

.. code-block:: ruby

    class SomeOutput < Fluent::BufferedOutput
      # Register plugin first. NAME is the name of this plugin
      # which is used in the configuration file.
      Fluent::Plugin.register_output('NAME', self)

      # This method is called before starting.
      # 'conf' is a Hash that includes configuration parameters.
      # If the configuration is invalid, raise Fluent::ConfigError.
      def configure(conf)
        super
        @path = conf['path']
        ...
      end

      # This method is called when starting.
      # Open sockets or files here.
      def start
        super
        ...
      end

      # This method is called when shutting down.
      # Shutdown the thread and close sockets or files here.
      def shutdown
        super
        ...
      end

      # This method is called when an event is reached.
      # Convert event to a raw string.
      def format(tag, time, record)
        [tag, time, record].to_json + "\n"
      end

      ## optionally, you can use to_msgpack to serialize the object.
      #def format(tag, time, record)
      #  [tag, time, record].to_msgpack
      #end

      # This method is called every flush interval. write the buffer chunk
      # to files or databases here.
      # 'chunk' is a buffer chunk that includes multiple formatted
      # events. You can use 'data = chunk.read' to get all events and
      # 'chunk.open {|io| ... }' to get IO object.
      def write(chunk)
        data = chunk.read
        print data
      end

      ## optionally, you can use chunk.msgpack_each to deserialize objects.
      #def write(chunk)
      #  chunk.msgpack_each {|(tag,time,record)|
      #  }
      #end
    end


Time sliced output plugins
------------------------------------

Time sliced output plugins are extended version of buffered output plugin. One of the examples of time sliced output is ``out_file`` plugin.

Note that it uses file buffer by default. Thus ``buffer_path`` option is required.

To implement time sliced output plugin, Extend **Fluent::TimeSlicedOutput** class and implement following methods.

.. code-block:: ruby

    class SomeOutput < Fluent::TimeSlicedOutput
      # configure(conf), start(), shutdown() and format(tag, time, record) are
      # same as BufferedOutput.
      ...

      # You can use 'chunk.key' to get sliced time. Format of the 'chunk.key'
      # can be configured by 'time_format' option. Default format is %Y%m%d.
      def write(chunk)
        day = chunk.key
        ...
      end
    end


Non-buffered output plugins
------------------------------------

Extend **Fluent::Output** class and implement following methods.

.. code-block:: ruby

    class SomeOutput < Fluent::Output
      # Register plugin first. NAME is the name of this plugin
      # which is used in the configuration file.
      Fluent::Plugin.register_output('NAME', self)

      # This method is called before starting.
      def configure(conf)
        super
        ...
      end
    
      # This method is called when starting.
      def start
        super
        ...
      end
    
      # This method is called when shutting down.
      def shutdown
        super
        ...
      end
    
      # This method is called when an event is reached.
      # 'es' is a Fluent::EventStream object that includes multiple events.
      # You can use 'es.each {|time,record| ... }' to retrieve events.
      # 'chain' is an object that manages transaction. Call 'chain.next' at
      # appropriate point and rollback if it raises exception.
      def emit(tag, es, chain)
        chain.next
        es.each {|time,record|
          $stderr.puts "OK!"
        }
      end
    end


Buffer plugins
------------------------------------

TODO


Custom parser for Tail input plugin
------------------------------------

You can customize text parser of Tail input plugin by extending **Fluent::TailInput** class.

Put following file to **/etc/fluent/plugin/in_mytail.rb**.

.. code-block:: ruby

    class MyTailInput < Fluent::TailInput
      Fluent::Plugin.register_input('mytail', self)
    
      # Override 'configure_parser(conf)' method.
      # You can get config parameters in this method.
      def configure_parser(conf)
        @time_format = conf['time_format'] || '%Y-%M-%d %H:%M:%S'
      end
    
      # Override 'parse_line(line)' method that returns time and record.
      # This example method assumes following log format:
      #   %Y-%m-%d %H:%M:%S\tkey1\tvalue1\tkey2\tvalue2...
      #   %Y-%m-%d %H:%M:%S\tkey1\tvalue1\tkey2\tvalue2...
      #   ...
      def parse_line(line)
        elements = line.split("\t")
        
        time = elements.shift
        time = Time.strptime(time, @time_format).to_i
        
        # [k1, v1, k2, v2, ...] -> {k1=>v1, k2=>v2, ...}
        record = {}
        while (k = elements.shift) && (v = elements.shift)
          record[k] = v
        end
        
        return time, record
      end
    end

Use following configuration file::

    <source>
      type mytail
      path /path/to/myformat_file
      tag myapp.mytail
    </source>


Debugging plugins
------------------------------------

Run ``fluentd`` with ``-vv`` option to show debug messages::

    $ fluentd -vv

**stdout** and **copy** output plugins are useful for debugging. **stdout** output plugin dumps matched events to the console. It can be used as following::

    # You want to debug this plugin
    <source>
      type your_custom_input_plugin
    </source>

    # Dump all events to stdout
    <match **>
      type stdout
    </match>

**copy** output plugin copies matched events to multiple output plugins. You can use it with the stdout plugin::

    <source>
      type tcp
    </source>
    # Use tcp input plugin and fluent-cat command to feed events:
    #  $ echo '{"event":"message"}' | fluent-cat test.tag

    <match test.tag>
      type copy

      # Dump the matched events
      <store>
        type stdout
      </store>

      # And feed them to your plugin
      <store>
        type your_custom_output_plugin
      </store>
    </match>


Writing test cases
------------------------------------

Fluentd provides unit test frameworks for plugins:

  Fluent::Test::InputTestDriver
    Test driver for input plugins.

  Fluent::Test::BufferedOutputTestDriver
    Test driver for buffered output plugins.

  Fluent::Test::OutputTestDriver
    Test driver for non-buffered output plugins.

See fluentd's source code for details.



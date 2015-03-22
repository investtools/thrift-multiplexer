# Thrift::Multiplexer

This gem implements multiplexing for [Apache Thrift](https://thrift.apache.org/).

For more information on Multiplexing Services, please access [THRIFT-1915](https://issues.apache.org/jira/browse/THRIFT-1915).

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'thrift-multiplexer'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install thrift-multiplexer

## Usage

### Client

```ruby
$:.push('gen-rb')
$:.unshift '../../lib/rb/lib'

require 'thrift'
require 'thrift/multiplexer'

require 'calculator'


begin
  port = ARGV[0] || 9090

  transport = Thrift::BufferedTransport.new(Thrift::Socket.new('127.0.0.1', port))
  protocol = Thrift::MultiplexedProtocol.new(Thrift::BinaryProtocol.new(transport), 'Abc')
  client = Calculator::Client.new(protocol)
  protocol2 = Thrift::MultiplexedProtocol.new(Thrift::BinaryProtocol.new(transport), 'Xyz')
  client2 = Calculator::Client.new(protocol)

  transport.open()

  client.ping()
  print "ping()\n"

  sum = client2.add(1,1)
  print "1+1=", sum, "\n"

  sum = client2.add(1,4)
  print "1+4=", sum, "\n"

  work = Work.new()

  work.op = Operation::SUBTRACT
  work.num1 = 15
  work.num2 = 10
  diff = client.calculate(1, work)
  print "15-10=", diff, "\n"

  log = client.getStruct(1)
  print "Log: ", log.value, "\n"

  begin
    work.op = Operation::DIVIDE
    work.num1 = 1
    work.num2 = 0
    quot = client.calculate(1, work)
    puts "Whoa, we can divide by 0 now?"
  rescue InvalidOperation => io
    print "InvalidOperation: ", io.why, "\n"
  end

  client.zip()
  print "zip\n"

  transport.close()

rescue Thrift::Exception => tx
  print 'Thrift::Exception: ', tx.message, "\n"
end
```

### Server

```ruby
$:.push('gen-rb')
$:.unshift '../../lib/rb/lib'

require 'thrift'
require 'thrift/multiplexer'

require 'calculator'
require 'shared_types'

class CalculatorHandler
  def initialize()
    @log = {}
  end

  def ping()
    puts "ping()"
  end

  def add(n1, n2)
    print "add(", n1, ",", n2, ")\n"
    return n1 + n2
  end

  def calculate(logid, work)
    print "calculate(", logid, ", {", work.op, ",", work.num1, ",", work.num2,"})\n"
    if work.op == Operation::ADD
      val = work.num1 + work.num2
    elsif work.op == Operation::SUBTRACT
      val = work.num1 - work.num2
    elsif work.op == Operation::MULTIPLY
      val = work.num1 * work.num2
    elsif work.op == Operation::DIVIDE
      if work.num2 == 0
        x = InvalidOperation.new()
        x.what = work.op
        x.why = "Cannot divide by 0"
        raise x
      end
      val = work.num1 / work.num2
    else
      x = InvalidOperation.new()
      x.what = work.op
      x.why = "Invalid operation"
      raise x
    end

    entry = SharedStruct.new()
    entry.key = logid
    entry.value = "#{val}"
    @log[logid] = entry

    return val

  end

  def getStruct(key)
    print "getStruct(", key, ")\n"
    return @log[key]
  end

  def zip()
    print "zip\n"
  end

end

handler = CalculatorHandler.new()
processor = Thrift::MultiplexedProcessor.new
processor.register_processor('Abc', Calculator::Processor.new(handler))
processor.register_processor('Xyz', Calculator::Processor.new(handler))
transport = Thrift::ServerSocket.new(9090)
transportFactory = Thrift::BufferedTransportFactory.new()
server = Thrift::SimpleServer.new(processor, transport, transportFactory)

puts "Starting the server..."
server.serve()
puts "done."
```

## Contributing

1. Fork it ( https://github.com/investtools/thrift-multiplexer/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request

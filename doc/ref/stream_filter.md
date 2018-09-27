### jsoncons::stream_filter

```c++
typedef basic_stream_filter<char> stream_filter
```

#### Header
```c++
#include <jsoncons/stream_reader.hpp>
```

#### Destructor

    virtual ~basic_stream_filter() = default;

#### Member functions

    virtual bool accept(const basic_stream_event<CharT>& event) = 0;
Tests whether the [current event](stream_event.md) is part of the stream. Returns `true` if the filter accepts the event, `false` otherwise.


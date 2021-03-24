# Evaluating JavaScript Performance

## The current implementation

* on page load rooms-data-controller#update is called, which:
  * dispatches a `tallies-updated` event from each of its `tallyTargets` (filter checkbox)
    * there are 63 targets resulting in 63 events
    * each checkbox is listening for that event calculates a tally for the filter using data in the event

https://cacoo.com/diagrams/4O2MSsQavwBqz8Qc/73BF7

How expensive are these events?

## Using the JavaScript performance interface

https://developer.mozilla.org/en-US/docs/Web/API/Performance

Add some code to `rooms_data_controller.js`:

```javascript
update(event) {
  // call performance.now() to capture the start time
  const perfStart = performance.now();

  const talliesUpdated = new CustomEvent("tallies-updated", {
    bubbles: true,
    detail: {
      visibleRooms: this.filterDataSets
    }
  });

  this.tallyTargets.forEach(tallyTarget => tallyTarget.dispatchEvent(talliesUpdated));

  // call performance.now() to capture the end time
  const perfEnd = performance.now();


  // use console.log to render a message to the console
  console.log(`$rooms_data_controller#update with ${event.type} took ${(perfEnd - perfStart).toFixed(4)}, milliseconds to run`);

  // log some memory info as well:
  console.log(`rooms_data_controller used ${performance.memory.usedJSHeapSize} bytes of memory out of ${performance.memory.jsHeapSizeLimit}`);
}
```

## Update the Rails test environment to write JavaScript console logs:

https://github.umn.edu/asrweb/roomsearch/blob/b42cc21e2a16521bcd5817ffb0a21e43713729cc/spec/helpers.rb

Add a test helper:

```ruby
module Helpers
  def write_console_log_to_file
    logs = page.driver.browser.manage.logs.get(:browser)
    console_log = "#{logs.map(&:message).join("\n")}\n"
    file_name =  "#{self.class.to_s.split("::").drop(2).join("-").downcase}-#{Time.now.to_i}.log"
    dir = Rails.root.join("tmp", "console_logs")
    Dir.mkdir(dir) unless Dir.exist?(dir)
    File.write(Rails.root.join("tmp", "console_logs", file_name), console_log)
  end
end
```

Update `ApplicationSystemTestCase`

https://github.umn.edu/asrweb/roomsearch/blob/b42cc21e2a16521bcd5817ffb0a21e43713729cc/spec/application_system_test_case.rb

```ruby
# require the helper
require "helpers"

# update the Chrome configuration
caps = Selenium::WebDriver::Remote::Capabilities.chrome("goog:loggingPrefs": { browser: 'ALL' })

 config.include Helpers
```

Update a system test to write the JavaScript console logs after each test:

https://github.umn.edu/asrweb/roomsearch/blob/b42cc21e2a16521bcd5817ffb0a21e43713729cc/spec/system/filter_tally_spec.rb#L8-L10

```ruby
after(:each) do
  write_console_log_to_file
end
```

### Log with events:

Running a system test that checks the state on page load three times gives us this information:

```sh
http://web:3001/packs-test/js/application-5b7077180dab2b3d7a0c.js 1324:14 "rooms_data_controller#update with load took 340.1400, milliseconds run"
http://web:3001/packs-test/js/application-5b7077180dab2b3d7a0c.js 1329:14 "rooms_data_controller used 7079429 bytes of memory out of 4294705152"

http://web:3001/packs-test/js/application-b92e4eab558ea08a6d44.js 1324:14 "rooms_data_controller#update with load took 344.3100, milliseconds to run"
http://web:3001/packs-test/js/application-b92e4eab558ea08a6d44.js 1329:14 "rooms_data_controller used 7097513 bytes of memory out of 4294705152"

http://web:3001/packs-test/js/application-b92e4eab558ea08a6d44.js 1324:14 "rooms_data_controller#update with load took 362.3200, milliseconds to run"
http://web:3001/packs-test/js/application-b92e4eab558ea08a6d44.js 1329:14 "rooms_data_controller used 7148294 bytes of memory out of 4294705152"
```

## Rewrite the code to use Stimulus targets instead of passing events around

[This code could use some cleanup, but it's functional](https://github.umn.edu/asrweb/roomsearch/compare/js_perf_spike_targets).

### Log with targets:

Running the same system test that checks the state on page load three times gives us this information:

```sh
http://web:3001/packs-test/js/application-e3fbc363ac8baf0a125c.js 1356:14 "rooms_data_controller#update with load took 228.2650, milliseconds to run"
http://web:3001/packs-test/js/application-e3fbc363ac8baf0a125c.js 1361:14 "rooms_data_controller used 6293876 bytes of memory out of 4294705152"

http://web:3001/packs-test/js/application-e3fbc363ac8baf0a125c.js 1356:14 "rooms_data_controller#update with load took 234.6850, milliseconds to run"
http://web:3001/packs-test/js/application-e3fbc363ac8baf0a125c.js 1361:14 "rooms_data_controller used 6226487 bytes of memory out of 4294705152"

http://web:3001/packs-test/js/application-e3fbc363ac8baf0a125c.js 1356:14 "rooms_data_controller#update with load took 230.5550, milliseconds to run"
http://web:3001/packs-test/js/application-e3fbc363ac8baf0a125c.js 1361:14 "rooms_data_controller used 6311944 bytes of memory out of 4294705152"
```

### Comparison

| method  | ms to run  | memory (bytes) | memory (mb) |
|---------|------------|----------------|-------------|
| events  | 340.140    | 7079429        | 7.08        |
| events  | 344.310    | 7097513        | 7.10        |
| events  | 362.320    | 7148294        | 7.15        |
| targets | 228.265    | 6293876        | 6.29        |
| targets | 234.685    | 6226487        | 6.23        |
| targets | 230.555    | 6311944        | 6.31        |

## Conclusions

* This is running in test environment that only has 36 rooms to evaluate. Production has over 500, so the performance hit is likely worse.
* In this case targets outperform events both with memory use and are approximately 30% faster.

# envconfig

```Go
import "github.com/kelseyhightower/envconfig"
```

## Documentation

See [godoc](http://godoc.org/github.com/kelseyhightower/envconfig)

## Usage

Set some environment variables:

```Bash
export MYAPP_DEBUG=false
export MYAPP_PORT=8080
export MYAPP_USER=Kelsey
export MYAPP_RATE="0.5"
export MYAPP_TIMEOUT="3m"
export MYAPP_USERS="rob,ken,robert"
export MYAPP_COLORCODES="red:1,green:2,blue:3"
```

Write some code:

```Go
package main

import (
    "fmt"
    "log"
    "time"

    "github.com/kelseyhightower/envconfig"
)

type Specification struct {
    Debug       bool
    Port        int
    User        string
    Users       []string
    Rate        float32
    Timeout     time.Duration
    ColorCodes  map[string]int
}

func main() {
    var s Specification
    err := envconfig.Process("myapp", &s)
    if err != nil {
        log.Fatal(err.Error())
    }
    format := "Debug: %v\nPort: %d\nUser: %s\nRate: %f\nTimeout: %s\n"
    _, err = fmt.Printf(format, s.Debug, s.Port, s.User, s.Rate, s.Timeout)
    if err != nil {
        log.Fatal(err.Error())
    }

    fmt.Println("Users:")
    for _, u := range s.Users {
        fmt.Printf("  %s\n", u)
    }

    fmt.Println("Color codes:")
    for k, v := range s.ColorCodes {
        fmt.Printf("  %s: %d\n", k, v)
    }
}
```

Results:

```Bash
Debug: false
Port: 8080
User: Kelsey
Rate: 0.500000
Timeout: 3m0s
Users:
  rob
  ken
  robert
Color codes:
  red: 1
  green: 2
  blue: 3
```

## Struct Tag Support

Envconfig supports the use of struct tags to specify alternate, default, and required
environment variables.

For example, consider the following struct:

```Go
type Specification struct {
    ManualOverride1 string `envconfig:"manual_override_1"`
    DefaultVar      string `default:"foobar"`
    RequiredVar     string `required:"true"`
    IgnoredVar      string `ignored:"true"`
    AutoSplitVar    string `split_words:"true"`
    RequiredAndAutoSplitVar    string `required:"true" split_words:"true"`
}
```

Envconfig has automatic support for CamelCased struct elements when the
`split_words:"true"` tag is supplied. Without this tag, `AutoSplitVar` above
would look for an environment variable called `MYAPP_AUTOSPLITVAR`. With the
setting applied it will look for `MYAPP_AUTO_SPLIT_VAR`. Note that numbers
will get globbed into the previous word. If the setting does not do the
right thing, you may use a manual override.

Envconfig will process value for `ManualOverride1` by populating it with the
value for `MYAPP_MANUAL_OVERRIDE_1`. Without this struct tag, it would have
instead looked up `MYAPP_MANUALOVERRIDE1`. With the `split_words:"true"` tag
it would have looked up `MYAPP_MANUAL_OVERRIDE1`.

```Bash
export MYAPP_MANUAL_OVERRIDE_1="this will be the value"

# export MYAPP_MANUALOVERRIDE1="and this will not"
```

If envconfig can't find an environment variable value for `MYAPP_DEFAULTVAR`,
it will populate it with "foobar" as a default value.

If envconfig can't find an environment variable value for `MYAPP_REQUIREDVAR`,
it will return an error when asked to process the struct.  If
`MYAPP_REQUIREDVAR` is present but empty, envconfig will not return an error.

If envconfig can't find an environment variable in the form `PREFIX_MYVAR`, and there
is a struct tag defined, it will try to populate your variable with an environment
variable that directly matches the envconfig tag in your struct definition:

```shell
export SERVICE_HOST=127.0.0.1
export MYAPP_DEBUG=true
```
```Go
type Specification struct {
    ServiceHost string `envconfig:"SERVICE_HOST"`
    Debug       bool
}
```

Envconfig won't process a field with the "ignored" tag set to "true", even if a corresponding
environment variable is set.

## Supported Struct Field Types

envconfig supports these struct field types:

  * string
  * int8, int16, int32, int64
  * bool
  * float32, float64
  * slices of any supported type
  * maps (keys and values of any supported type)
  * [encoding.TextUnmarshaler](https://golang.org/pkg/encoding/#TextUnmarshaler)
  * [encoding.BinaryUnmarshaler](https://golang.org/pkg/encoding/#BinaryUnmarshaler)
  * [time.Duration](https://golang.org/pkg/time/#Duration)

Embedded structs using these fields are also supported.

## Custom Decoders

Any field whose type (or pointer-to-type) implements `envconfig.Decoder` can
control its own deserialization:

```Bash
export DNS_SERVER=8.8.8.8
```

```Go
type IPDecoder net.IP

func (ipd *IPDecoder) Decode(value string) error {
    *ipd = IPDecoder(net.ParseIP(value))
    return nil
}

type DNSConfig struct {
    Address IPDecoder `envconfig:"DNS_SERVER"`
}
```

Example for decoding the environment variables into map[string][]structName type

```Bash
export SMS_PROVIDER_WITH_WEIGHT= `IND=[{"name":"SMSProvider1","weight":70},{"name":"SMSProvider2","weight":30}];US=[{"name":"SMSProvider1","weight":100}]`
```

```GO
type providerDetails struct {
	Name   string
	Weight int
}

type SMSProviderDecoder map[string][]providerDetails

func (sd *SMSProviderDecoder) Decode(value string) error {
	smsProvider := map[string][]providerDetails{}
	pairs := strings.Split(value, ";")
	for _, pair := range pairs {
		providerdata := []providerDetails{}
		kvpair := strings.Split(pair, "=")
		if len(kvpair) != 2 {
			return fmt.Errorf("invalid map item: %q", pair)
		}
		err := json.Unmarshal([]byte(kvpair[1]), &providerdata)
		if err != nil {
			return fmt.Errorf("invalid map json: %w", err)
		}
		smsProvider[kvpair[0]] = providerdata

	}
	*sd = SMSProviderDecoder(smsProvider)
	return nil
}

type SMSProviderConfig struct {
    ProviderWithWeight SMSProviderDecoder `envconfig:"SMS_PROVIDER_WITH_WEIGHT"`
}
```

Also, envconfig will use a `Set(string) error` method like from the
[flag.Value](https://godoc.org/flag#Value) interface if implemented.

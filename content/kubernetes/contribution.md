---
title: "Contribution"
date: 2020-02-22T13:44:12+08:00
draft: true
---

### general

code organized by SIGs (sepcial interest gruop)
wg (working group) is across different SIGs

### wg-component-standard

Standardize components
* Config and flags
* Logging
* Monitoring and debugging endpoints


config always start with a config file, all components should support read from config file.
kubelet and kube-proxy might consume data in a configmap through API.

components command line flags are not easy to version, instead, use a config file with a version schema.

github.com/kubernetes-sig/legacyflag
encoder/decoder, decoder test suite
kubebuilder, generate component config and legacyflag
multi-doc support


1. think of a test strategey
2. understand kubelet bootstrap


### migration
`<components>/cmd/options.go`
write the value into the component cofnig structure.
kube-proxy, kube-controller-manager have more complicate flags and configuration so it's not a good example for migration.
pkg/apis/config


```
cmd/kubelet/app/options/options.go
pkg/kubelet/apis/config/scheme/scheme.go
pkg/kubelet/apis/config/v1beta1 // this is the external type
pkg/kubelet/apis/config // this is the internal type
```
internal type is what's the component wants to get in memory
read an external type from disk, and deserialize into an internal type


### runtime
gvk stands for group, version, kind

Scheme defines methods for serializing and deserializing API objects
Converter convert one type to another

check conversition, why use pointer to interface?



### networking
https://en.wikipedia.org/wiki/Hairpinning

### architecture
https://en.wikipedia.org/wiki/Non-uniform_memory_access

### questions?
1. who control which Kind in config file?
2. who will read the config file?
3. how the flag and config file merge together?
4. who print the deperated information?

### common pattern

```go
type SchemeBuilder []func(*Scheme) error

// AddToScheme applies all the stored functions to the scheme. A non-nil error
// indicates that one function failed and the attempt was abandoned.
func (sb *SchemeBuilder) AddToScheme(s *Scheme) error {
	for _, f := range *sb {
		if err := f(s); err != nil {
			return err
		}
	}
	return nil
}

// Register adds a scheme setup function to the list.
func (sb *SchemeBuilder) Register(funcs ...func(*Scheme) error) {
	for _, f := range funcs {
		*sb = append(*sb, f)
	}
}

// NewSchemeBuilder calls Register for you.
func NewSchemeBuilder(funcs ...func(*Scheme) error) SchemeBuilder {
	var sb SchemeBuilder
	sb.Register(funcs...)
	return sb
}
```

### communication

just add a little bit color to that*
getting into the weeds*
require a mount love*



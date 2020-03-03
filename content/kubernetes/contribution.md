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

```
pkg/kubelet/apis/config/v1beta1/defaults.go // this is how the default value get filled in internal type
staging/src/k8s.io/kubelet/config/v1beta1/types.go // this is the external type, it has pointer and optional value
pkg/kubelet/apis/config/types.go // this is internal type
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










### kubelet

cmd/kubelet/kubelet.go
```go
func main() {
	rand.Seed(time.Now().UnixNano())

	command := app.NewKubeletCommand() // 创建 kubelet 的 cobra command
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

cmd/kubelet/app/server.go
```go
func NewKubeletCommand() *cobra.Command {
	cleanFlagSet := pflag.NewFlagSet(componentKubelet, pflag.ContinueOnError)
	cleanFlagSet.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	kubeletFlags := options.NewKubeletFlags() // 创建 KubeletFlags struct 
	kubeletConfig, err := options.NewKubeletConfiguration() // 创建 KubeletConfiguration struct 
	// programmer error
	if err != nil {
		klog.Fatal(err)
	}

	cmd := &cobra.Command{
  // 初始化 cobra command
  // ...
  }
  
  return cmd
}
```

cmd/kubelet/app/options/options.go
```go
func NewKubeletConfiguration() (*kubeletconfig.KubeletConfiguration, error) {
	scheme, _, err := kubeletscheme.NewSchemeAndCodecs() // 工具方法创建 scheme 和 codecs
	if err != nil {
		return nil, err
	}
	versioned := &v1beta1.KubeletConfiguration{}
	scheme.Default(versioned)
	config := &kubeletconfig.KubeletConfiguration{}
	if err := scheme.Convert(versioned, config, nil); err != nil {
		return nil, err
	}
	applyLegacyDefaults(config)
	return config, nil
}
```

scheme 是用来序列化和反序列化 API Object，也是处理不同版本 API 转化的核心逻辑。

pkg/kubelet/apis/config/scheme/scheme.go
```go
func NewSchemeAndCodecs(mutators ...serializer.CodecFactoryOptionsMutator) (*runtime.Scheme, *serializer.CodecFactory, error) {
	scheme := runtime.NewScheme() // 创建 scheme
	if err := kubeletconfig.AddToScheme(scheme); err != nil { // 注册内部的 type，在这里是 KubeletConfiguration 和 SerializedNodeConfigSource
		return nil, nil, err
	}
	if err := kubeletconfigv1beta1.AddToScheme(scheme); err != nil { // 注册 staging type，在这里是 KubeletConfiguration 和 SerializedNodeConfigSource 并且注册了默认值 (pkg)
		return nil, nil, err
	}
	codecs := serializer.NewCodecFactory(scheme, mutators...)
	return scheme, &codecs, nil
}
```

pkg/runtime/scheme.go
```go
func NewScheme() *Scheme {
	s := &Scheme{
		gvkToType:                 map[schema.GroupVersionKind]reflect.Type{}, // api version 对应的内部 type
		typeToGVK:                 map[reflect.Type][]schema.GroupVersionKind{}, // 内部 type 对应多个 api version
		unversionedTypes:          map[reflect.Type]schema.GroupVersionKind{}, // 内部 type 对应的内部 api version
		unversionedKinds:          map[string]reflect.Type{}, // 
		fieldLabelConversionFuncs: map[schema.GroupVersionKind]FieldLabelConversionFunc{}, // api version 对应的序列化 label
		defaulterFuncs:            map[reflect.Type]func(interface{}){},
		versionPriority:           map[string][]string{},
		schemeName:                naming.GetNameFromCallsite(internalPackages...),
	}
	s.converter = conversion.NewConverter(s.nameFunc) // nameFunc 通过 type 返回kind

	// Enable couple default conversions by default.
	utilruntime.Must(RegisterEmbeddedConversions(s)) // 注册 Object 与 RawExtension 的转换

	// 注册了以下类型变换 
	// Convert_Slice_string_To_string
	// Convert_Slice_string_To_int
	// Convert_Slice_string_To_bool
	// Convert_Slice_string_To_int64
	utilruntime.Must(RegisterStringConversions(s)) 

	utilruntime.Must(s.RegisterInputDefaults(&map[string][]string{}, JSONKeyMapper, conversion.AllowDifferentFieldTypeNames|conversion.IgnoreMissingFields))
	utilruntime.Must(s.RegisterInputDefaults(&url.Values{}, JSONKeyMapper, conversion.AllowDifferentFieldTypeNames|conversion.IgnoreMissingFields))
	return s
}
```

```go
func (s *Scheme) nameFunc(t reflect.Type) string {
	// find the preferred names for this type
	gvks, ok := s.typeToGVK[t]
	if !ok {
		return t.Name()
	}

	for _, gvk := range gvks { // 问题？为什么要 range 选择[0]的 group 不行吗？
		internalGV := gvk.GroupVersion() // 返回的是全新的 gv，所以下面不会覆盖
		internalGV.Version = APIVersionInternal // 变成 version 变成内部
		internalGVK := internalGV.WithKind(gvk.Kind)

		if internalType, exists := s.gvkToType[internalGVK]; exists { // 找到 type 对应的内部 GVK 的 kind 并返回
			return s.typeToGVK[internalType][0].Kind
		}
	}

	return gvks[0].Kind // 返回第一个 kind
}
```

```go
type typePair struct {
	source reflect.Type
	dest   reflect.Type
}

type typeNamePair struct {
	fieldType reflect.Type
	fieldName string
}

type ConversionFunc func(a, b interface{}, scope Scope) error

type ConversionFuncs struct {
	untyped map[typePair]ConversionFunc
}

type FieldMappingFunc func(key string, sourceTag, destTag reflect.StructTag) (source string, dest string)

type FieldMatchingFlags int

func NewConverter(nameFn NameFunc) *Converter {
	c := &Converter{
		conversionFuncs:           NewConversionFuncs(),
		generatedConversionFuncs:  NewConversionFuncs(),
		ignoredConversions:        make(map[typePair]struct{}),
		ignoredUntypedConversions: make(map[typePair]struct{}),
		nameFunc:                  nameFn,
		structFieldDests:          make(map[typeNamePair][]typeNamePair),
		structFieldSources:        make(map[typeNamePair][]typeNamePair),

		inputFieldMappingFuncs: make(map[reflect.Type]FieldMappingFunc),
		inputDefaultFlags:      make(map[reflect.Type]FieldMatchingFlags),
	}
	c.RegisterUntypedConversionFunc(
		(*[]byte)(nil), (*[]byte)(nil),
		func(a, b interface{}, s Scope) error {
			return Convert_Slice_byte_To_Slice_byte(a.(*[]byte), b.(*[]byte), s)
		},
	)
	return c
}
```

```go
func (c *Converter) RegisterUntypedConversionFunc(a, b interface{}, fn ConversionFunc) error {
	return c.conversionFuncs.AddUntyped(a, b, fn) // 如果是指针 就加入 map
}
```

内部struct pkg/kubelet/apis/config/register.go and pkg/kubelet/apis/config/types.go
v1beta1 默认值 pkg/kubelet/apis/config/v1beta1/defaults.go
v1beta1 struct staging/src/k8s.io/kubelet/config/v1beta1/register.go and staging/src/k8s.io/kubelet/config/v1beta1/types.go

用 builder 方式注册不同的 config 到 scheme

pkg/runtime/scheme_builder.go
```go
type SchemeBuilder []func(*Scheme) error

func (sb *SchemeBuilder) AddToScheme(s *Scheme) error { // 执行所有注册的 build scheme 的方法
	for _, f := range *sb { 
		if err := f(s); err != nil {
			return err
		}
	}
	return nil
}

func (sb *SchemeBuilder) Register(funcs ...func(*Scheme) error) { // 注册scheme build 的方法
	for _, f := range funcs {
		*sb = append(*sb, f)
	}
}

func NewSchemeBuilder(funcs ...func(*Scheme) error) SchemeBuilder {
	var sb SchemeBuilder
	sb.Register(funcs...)
	return sb
}
```

staging/src/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go
implement `staging/src/k8s.io/apimachinery/pkg/runtime/interfaces.go` Decoder 接口

```go
func (s *Serializer) Decode(originalData []byte, gvk *schema.GroupVersionKind, into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {
	data := originalData
	if s.options.Yaml {
		altered, err := yaml.YAMLToJSON(data)
		if err != nil {
			return nil, nil, err
		}
		data = altered
	}

	actual, err := s.meta.Interpret(data)
	if err != nil {
		return nil, nil, err
	}

	if gvk != nil {
		*actual = gvkWithDefaults(*actual, *gvk)
	}

	if unk, ok := into.(*runtime.Unknown); ok && unk != nil {
		unk.Raw = originalData
		unk.ContentType = runtime.ContentTypeJSON
		unk.GetObjectKind().SetGroupVersionKind(*actual)
		return unk, actual, nil
	}

	if into != nil {
		_, isUnstructured := into.(runtime.Unstructured)
		types, _, err := s.typer.ObjectKinds(into)
		switch {
		case runtime.IsNotRegisteredError(err), isUnstructured:
			if err := caseSensitiveJsonIterator.Unmarshal(data, into); err != nil {
				return nil, actual, err
			}
			return into, actual, nil
		case err != nil:
			return nil, actual, err
		default:
			*actual = gvkWithDefaults(*actual, types[0])
		}
	}

	if len(actual.Kind) == 0 {
		return nil, actual, runtime.NewMissingKindErr(string(originalData))
	}
	if len(actual.Version) == 0 {
		return nil, actual, runtime.NewMissingVersionErr(string(originalData))
	}

	// use the target if necessary
	obj, err := runtime.UseOrCreateObject(s.typer, s.creater, *actual, into)
	if err != nil {
		return nil, actual, err
	}

	if err := caseSensitiveJsonIterator.Unmarshal(data, obj); err != nil {
		return nil, actual, err
	}

	// If the deserializer is non-strict, return successfully here.
	if !s.options.Strict {
		return obj, actual, nil
	}

	// In strict mode pass the data trough the YAMLToJSONStrict converter.
	// This is done to catch duplicate fields regardless of encoding (JSON or YAML). For JSON data,
	// the output would equal the input, unless there is a parsing error such as duplicate fields.
	// As we know this was successful in the non-strict case, the only error that may be returned here
	// is because of the newly-added strictness. hence we know we can return the typed strictDecoderError
	// the actual error is that the object contains duplicate fields.
	altered, err := yaml.YAMLToJSONStrict(originalData)
	if err != nil {
		return nil, actual, runtime.NewStrictDecodingError(err.Error(), string(originalData))
	}
	// As performance is not an issue for now for the strict deserializer (one has regardless to do
	// the unmarshal twice), we take the sanitized, altered data that is guaranteed to have no duplicated
	// fields, and unmarshal this into a copy of the already-populated obj. Any error that occurs here is
	// due to that a matching field doesn't exist in the object. hence we can return a typed strictDecoderError,
	// the actual error is that the object contains unknown field.
	strictObj := obj.DeepCopyObject()
	if err := strictCaseSensitiveJsonIterator.Unmarshal(altered, strictObj); err != nil {
		return nil, actual, runtime.NewStrictDecodingError(err.Error(), string(originalData))
	}
	// Always return the same object as the non-strict serializer to avoid any deviations.
	return obj, actual, nil
}
```

staging/src/k8s.io/apimachinery/pkg/runtime/serializer/codec_factory.go
```go
func NewCodecFactory(scheme *runtime.Scheme, mutators ...CodecFactoryOptionsMutator) CodecFactory {
	options := CodecFactoryOptions{Pretty: true}
	for _, fn := range mutators {
		fn(&options)
	}

	serializers := newSerializersForScheme(scheme, json.DefaultMetaFactory, options) // 注册 json yaml proto raw 序列化
	return newCodecFactory(scheme, serializers)
}

func newSerializersForScheme(scheme *runtime.Scheme, mf json.MetaFactory, options CodecFactoryOptions) []serializerType {
	jsonSerializer := json.NewSerializerWithOptions(
		mf, scheme, scheme,
		json.SerializerOptions{Yaml: false, Pretty: false, Strict: options.Strict},
	)
	jsonSerializerType := serializerType{
		AcceptContentTypes: []string{runtime.ContentTypeJSON},
		ContentType:        runtime.ContentTypeJSON,
		FileExtensions:     []string{"json"},
		EncodesAsText:      true,
		Serializer:         jsonSerializer,

		Framer:           json.Framer,
		StreamSerializer: jsonSerializer,
	}
	if options.Pretty {
		jsonSerializerType.PrettySerializer = json.NewSerializerWithOptions(
			mf, scheme, scheme,
			json.SerializerOptions{Yaml: false, Pretty: true, Strict: options.Strict},
		)
	}

	yamlSerializer := json.NewSerializerWithOptions(
		mf, scheme, scheme,
		json.SerializerOptions{Yaml: true, Pretty: false, Strict: options.Strict},
	)
	protoSerializer := protobuf.NewSerializer(scheme, scheme)
	protoRawSerializer := protobuf.NewRawSerializer(scheme, scheme)

	serializers := []serializerType{
		jsonSerializerType,
		{
			AcceptContentTypes: []string{runtime.ContentTypeYAML},
			ContentType:        runtime.ContentTypeYAML,
			FileExtensions:     []string{"yaml"},
			EncodesAsText:      true,
			Serializer:         yamlSerializer,
		},
		{
			AcceptContentTypes: []string{runtime.ContentTypeProtobuf},
			ContentType:        runtime.ContentTypeProtobuf,
			FileExtensions:     []string{"pb"},
			Serializer:         protoSerializer,

			Framer:           protobuf.LengthDelimitedFramer,
			StreamSerializer: protoRawSerializer,
		},
	}

	for _, fn := range serializerExtensions {
		if serializer, ok := fn(scheme); ok {
			serializers = append(serializers, serializer)
		}
	}
	return serializers
}
```

pkg/kubelet/kubeletconfig/util/codec/codec.go
```go

```




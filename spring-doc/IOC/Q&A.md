# Q&A

## PathMatchingResourcePatternResolver 和 DefaultResourceLoader的getResource方法比较

## XML解析中的Questions

### 1:解析过程

- 配置文件读取，将xml文件读取为resource
- 资源的一些配置，比如忽略一些自动装配的接口，包括BeanNameAware，BeanFactoryAware，BeanClassLoaderAware。通过EncodedResource的编码处理
- 读取文件类型，是DTD还是XSD，默认XSD，判断方式为看文件有没有DOCTYPE
- 将inputsource解析成Document，其中EntityResolver作用是根据配置文件指定的publicId和SystemId寻找验证文件。
- 根据Document的Root元素读取属性并解析


### 2：Resource DTD XSD EntityResolver Element解析

- Resource 继承InputStreamSource， InputStreamSource接口只有一个getInputStream方法，获取流。 Resource的实现是抽象类AbstractResource，其子类众多，比如有ClassPathResource, EncodedResource,FileSystemResource等等，提供对不同文件的处理方式。
- DTD XSD是Spring支持xml配置文件验证的两种格式，其中XSD本身就是xml的格式，可以统一解析，默认的也是XSD。DTD的特征就是文件前面有< ! DOCTYPE 字眼。
- EntityResolver 是用来实现本地xml验证的，一般情况下，xml文件需要指定网站来对配置文件的格式经行验证，这样带来很多的局限性，比如网络验证时间长，离线无法进行之类的。EntityResolver 是可以指定本地的验证文件，在xml指定publicId和SystemId就可以找到对应的验证文件，一般DTD是根据"/"截取最后一个字符串，在当前路径下查找，XSD是在META-INF下查找验证文件
- Element解析
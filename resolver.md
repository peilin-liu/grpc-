使用示例：

int main(int argc, char** argv) {
  // Instantiate the client. It requires a channel, out of which the actual RPCs
  // are created. This channel models a connection to an endpoint (in this case,
  // localhost at port 50051). We indicate that the channel isn't authenticated
  // (use of InsecureChannelCredentials()).
  ChannelArguments args;
  // Set the load balancing policy for the channel.
  args.SetLoadBalancingPolicyName("round_robin");
  GreeterClient greeter(grpc::CreateCustomChannel(
      "ipv4:localhost:50051,localhost:50052,localhost:50053",
		  grpc::InsecureChannelCredentials(),
		  args));
  std::string user("world");
  std::string reply = greeter.SayHello(user);
  std::cout << "Greeter received: " << reply << std::endl;

  return 0;
}

这里我们使用 grpc::CreateCustomChannel 来创建一个ipv4 类型的解析器。

 	grpc_core::(anonymous namespace)::CreateChannel()	C++ (gdb)
 	grpc_channel_create()	C++ (gdb)
 	grpc::(anonymous namespace)::InsecureChannelCredentialsImpl::CreateChannelWithInterceptors()	C++ (gdb)
 	grpc::(anonymous namespace)::InsecureChannelCredentialsImpl::CreateChannelImpl()	C++ (gdb)
 	grpc::CreateCustomChannel()	C++ (gdb)
>	main()	C++ (gdb)

通过跟踪代码可以看到 	grpc_core::(anonymous namespace)::CreateChannel()	C++ (gdb) 会利用 
CoreConfiguration::Get().resolver_registry() 来获取解析器，并且尝试把uri 标准化。
然后通过 解析器注册的 scheme 来识别uri 并且调用到 SockaddrResolver 来解析服务器实例。
解析器都在 void BuildCoreConfiguration(CoreConfiguration::Builder* builder) 中注册。

我们可以看看解析器factory的基类设计
class ResolverFactory {
 public:
  virtual ~ResolverFactory() {}

  /// Returns the URI scheme that this factory implements.
  /// Caller does NOT take ownership of result.
  virtual absl::string_view scheme() const = 0;

  /// Returns a bool indicating whether the input uri is valid to create a
  /// resolver.
  virtual bool IsValidUri(const URI& uri) const = 0;

  /// Returns a new resolver instance.
  virtual OrphanablePtr<Resolver> CreateResolver(ResolverArgs args) const = 0;

  /// Returns a string representing the default authority to use for this
  /// scheme.
  virtual std::string GetDefaultAuthority(const URI& uri) const {
    return std::string(absl::StripPrefix(uri.path(), "/"));
  }
};

具体解析器的基类设计
class Resolver : public InternallyRefCounted<Resolver> {
 public:
  /// Results returned by the resolver.
  struct Result {
    /// A list of addresses, or an error.
    absl::StatusOr<ServerAddressList> addresses;
    /// A service config, or an error.
    absl::StatusOr<RefCountedPtr<ServiceConfig>> service_config = nullptr;
    /// An optional human-readable note describing context about the resolution,
    /// to be passed along to the LB policy for inclusion in RPC failure status
    /// messages in cases where neither \a addresses nor \a service_config
    /// has a non-OK status.  For example, a resolver that returns an empty
    /// address list but a valid service config may set to this to something
    /// like "no DNS entries found for <name>".
    std::string resolution_note;
    // TODO(roth): Before making this a public API, figure out a way to
    // avoid exposing channel args this way.
    ChannelArgs args;
  };
  从上我们看到这里使用工厂模式，通过接口 scheme 来定位具体需要的解析器。
  
  具体解析函数：
  bool ParseUri(const URI& uri,
              bool parse(const URI& uri, grpc_resolved_address* dst)
  
  从这里我们可以推理出，可以自己实现一套解析器，实现自定义的实例解析系统。
  
  但我们还缺少一套动态增删实例的方案，缺少一套自定义的负载均衡模式。

数据传输都是基于 channel 的。所以我们重点阅读clientchannel 的代码。

在clientchannel 中我们可以看到
void ClientChannel::LoadBalancedCall::PickSubchannel(void* arg,
                                                     grpc_error_handle error) {}
                                                     
被用来选择subchannel, 
>	grpc_core::LoadBalancingPolicy::QueuePicker::Pick()	C++ (gdb)
 	grpc_core::ClientChannel::LoadBalancedCall::PickSubchannelLocked()	C++ (gdb)
 	grpc_core::ClientChannel::LoadBalancedCall::PickSubchannel()	C++ (gdb)
 	grpc_core::ClientChannel::LoadBalancedCall::StartTransportStreamOpBatch()	C++ (gdb)
 	grpc_core::(anonymous namespace)::(anonymous namespace)::StartBatchInCallCombiner()	C++ (gdb)
 	exec_ctx_run()	C++ (gdb)
 	grpc_core::ExecCtx::Flush()	C++ (gdb)
 	grpc_core::ExecCtx::~ExecCtx()	C++ (gdb)
 	grpc_call_start_batch()	C++ (gdb)
 	grpc::CoreCodegen::grpc_call_start_batch()	C++ (gdb)
 	grpc::internal::CallOpSet<grpc::internal::CallOpSendInitialMetadata, grpc::internal::CallOpSendMessage, grpc::internal::CallOpRecvInitialMetadata, grpc::internal::CallOpRecvMessage<google::protobuf::MessageLite>, grpc::internal::CallOpClientSendClose, grpc::internal::CallOpClientRecvStatus>::ContinueFillOpsAfterInterception()	C++ (gdb)
 	grpc::internal::CallOpSet<grpc::internal::CallOpSendInitialMetadata, grpc::internal::CallOpSendMessage, grpc::internal::CallOpRecvInitialMetadata, grpc::internal::CallOpRecvMessage<google::protobuf::MessageLite>, grpc::internal::CallOpClientSendClose, grpc::internal::CallOpClientRecvStatus>::FillOps()	C++ (gdb)
 	grpc::Channel::PerformOpsOnCall()	C++ (gdb)
 	grpc::internal::Call::PerformOps()	C++ (gdb)
 	grpc::internal::BlockingUnaryCallImpl<google::protobuf::MessageLite, google::protobuf::MessageLite>::BlockingUnaryCallImpl()	C++ (gdb)
 	grpc::internal::BlockingUnaryCall<helloworld::HelloRequest, helloworld::HelloReply, google::protobuf::MessageLite, google::protobuf::MessageLite>()	C++ (gdb)
 	helloworld::Greeter::Stub::SayHello()	C++ (gdb)
 	GreeterClient::SayHello()	C++ (gdb)
 	main()	C++ (gdb)

io 线程
>	grpc_core::(anonymous namespace)::RoundRobin::Picker::Picker()	C++ (gdb)
 	std::make_unique<grpc_core::(anonymous namespace)::RoundRobin::Picker, grpc_core::(anonymous namespace)::RoundRobin*&, grpc_core::(anonymous namespace)::RoundRobin::RoundRobinSubchannelList*>()	C++ (gdb)
 	grpc_core::(anonymous namespace)::RoundRobin::RoundRobinSubchannelList::MaybeUpdateRoundRobinConnectivityStateLocked()	C++ (gdb)
 	grpc_core::(anonymous namespace)::RoundRobin::RoundRobinSubchannelData::ProcessConnectivityChangeLocked()	C++ (gdb)
 	grpc_core::SubchannelData<grpc_core::(anonymous namespace)::RoundRobin::RoundRobinSubchannelList, grpc_core::(anonymous namespace)::RoundRobin::RoundRobinSubchannelData>::Watcher::OnConnectivityStateChange()	C++ (gdb)
 	grpc_core::ClientChannel::SubchannelWrapper::WatcherWrapper::ApplyUpdateInControlPlaneWorkSerializer()	C++ (gdb)
 	grpc_core::ClientChannel::SubchannelWrapper::WatcherWrapper::OnConnectivityStateChange()::{lambda()#1}::operator()() const()	C++ (gdb)
 	std::_Function_handler<void (), grpc_core::ClientChannel::SubchannelWrapper::WatcherWrapper::OnConnectivityStateChange()::{lambda()#1}>::_M_invoke(std::_Any_data const&)()	C++ (gdb)
 	std::function<void ()>::operator()() const()	C++ (gdb)
 	grpc_core::WorkSerializer::WorkSerializerImpl::Run(std::function<void ()>, grpc_core::DebugLocation const&)()	C++ (gdb)
 	grpc_core::WorkSerializer::Run(std::function<void ()>, grpc_core::DebugLocation const&)()	C++ (gdb)
 	grpc_core::ClientChannel::SubchannelWrapper::WatcherWrapper::OnConnectivityStateChange()	C++ (gdb)
 	grpc_core::Subchannel::AsyncWatcherNotifierLocked::AsyncWatcherNotifierLocked(grpc_core::RefCountedPtr<grpc_core::Subchannel::ConnectivityStateWatcherInterface>, grpc_connectivity_state, absl::lts_20220623::Status const&)::{lambda(void*, absl::lts_20220623::Status)#1}::operator()(void*, absl::lts_20220623::Status) const()	C++ (gdb)
 	grpc_core::Subchannel::AsyncWatcherNotifierLocked::AsyncWatcherNotifierLocked(grpc_core::RefCountedPtr<grpc_core::Subchannel::ConnectivityStateWatcherInterface>, grpc_connectivity_state, absl::lts_20220623::Status const&)::{lambda(void*, absl::lts_20220623::Status)#1}::_FUN(void*, absl::lts_20220623::Status)()	C++ (gdb)
 	exec_ctx_run()	C++ (gdb)
 	grpc_core::ExecCtx::Flush()	C++ (gdb)
 	grpc_core::Executor::RunClosures()	C++ (gdb)
 	grpc_core::Executor::ThreadMain()	C++ (gdb)
 	grpc_core::(anonymous namespace)::ThreadInternalsPosix::<lambda(void*)>::operator()(void *) const()	C++ (gdb)
 	grpc_core::(anonymous namespace)::ThreadInternalsPosix::<lambda(void*)>::_FUN(void *)()	C++ (gdb)


然后我们再分析注册入口，以便使用相同方式注册我们自定义的负载均衡算法，我们可以看到在 
grpc_register_built_in_plugins 中使用有 grpc_register_plugin(grpc_lb_policy_round_robin_init, grpc_lb_policy_round_robin_shutdown);
void grpc_lb_policy_round_robin_init() {
  grpc_core::LoadBalancingPolicyRegistry::Builder::
      RegisterLoadBalancingPolicyFactory(
          absl::make_unique<grpc_core::RoundRobinFactory>());
}
注册 loadbanlance 插件。

看到这里，我们会明白需要进入 RoundRobinFactory 看他是如何工作的。

class RoundRobinConfig : public LoadBalancingPolicy::Config {
 public:
  absl::string_view name() const override { return kRoundRobin; }
};

class RoundRobinFactory : public LoadBalancingPolicyFactory {
 public:
  OrphanablePtr<LoadBalancingPolicy> CreateLoadBalancingPolicy(
      LoadBalancingPolicy::Args args) const override {
    return MakeOrphanable<RoundRobin>(std::move(args));
  }

  absl::string_view name() const override { return kRoundRobin; }

  absl::StatusOr<RefCountedPtr<LoadBalancingPolicy::Config>>
  ParseLoadBalancingConfig(const Json& /*json*/) const override {
    return MakeRefCounted<RoundRobinConfig>();
  }
};

在 RoundRobin::Picker 类中集成了具体的负载均衡算法， class Picker : public SubchannelPicker 可见我们需要查看SubchannelPicker 的接口定义。
  class SubchannelPicker {
   public:
    SubchannelPicker() = default;
    virtual ~SubchannelPicker() = default;

    virtual PickResult Pick(PickArgs args) = 0;
  };
  
  看到这里我，我们明白可以依葫芦画瓢实现自己的负载均衡算法。然后通过插件注册到框架里。

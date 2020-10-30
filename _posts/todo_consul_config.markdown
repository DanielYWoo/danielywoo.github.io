


# Consul Starter Initialization
The starter will register a ConsulPropertySourceLocator to spring context, later the Spring initialization flow will use PropertySourceBootstrapConfiguration
to call the locator to find the ConsulPropertSource.
```
public class ConsulConfigBootstrapConfiguration {
  ...
	protected static class ConsulPropertySourceConfiguration {
    ...
		@Bean
		public ConsulPropertySourceLocator consulPropertySourceLocator(
				ConsulConfigProperties consulConfigProperties) {
			return new ConsulPropertySourceLocator(this.consul, consulConfigProperties); --> This is the property source locator registered to Spring
		}
	}
}
```

# Initialize Configuration

Since ConsulPropertySource as a PropertySource, it will be loaded by Spring during initialization by the registered ConsulPropertySourceLocator. 
Then the properties are loaded by calling Consul in ConsulPropertySource.init().

```
	public void init() {
    ...
		Response<List<GetValue>> response = this.source.getKVValues(this.context,
				this.configProperties.getAclToken(), QueryParams.DEFAULT);

		this.initialIndex = response.getConsulIndex();

		final List<GetValue> values = response.getValue();
		ConsulConfigProperties.Format format = this.configProperties.getFormat();
		switch (format) {
		case KEY_VALUE:
			parsePropertiesInKeyValueFormat(values);
			break;
		case PROPERTIES:
		case YAML:
			parsePropertiesWithNonKeyValueFormat(values, format);
		}
	}

```

The ConsulePropertySourceLocator will try to locate 4 different locations in the order:
```
ConsulPropertySource {name='config/myApp,myProfile/'}
ConsulPropertySource {name='config/myApp/'}
ConsulPropertySource {name='config/application,myProfile/'}
ConsulPropertySource {name='config/application/'}
```

Following the order, the method CompositePropertySource.getProperty(name) will try to find the most specific key, if not found, try the next source.
```
	public Object getProperty(String name) {
		for (PropertySource<?> propertySource : this.propertySources) {
			Object candidate = propertySource.getProperty(name);
			if (candidate != null) {
				return candidate;
			}
		}
		return null;
	}

```

The 4 sources will be used to create a StandardEnvironment at the end of PropertySourceBootstrapConfiguration.initialize(appContext), by insertPropertySources(...). 
Then bind "spring.cloud.config"
```
  		Binder.get(environment(incoming)).bind("spring.cloud.config",
				Bindable.ofInstance(remoteProperties));
```

# Refresh Configuration

Config Watch
```
  @Override
  public void start() {
  if (this.running.compareAndSet(false, true)) {
    this.watchFuture = this.taskScheduler.scheduleWithFixedDelay(
        this::watchConfigKeyValues, this.properties.getWatch().getDelay());
  }


	@Timed("consul.watch-config-keys")
	public void watchConfigKeyValues() {
    ...
      RefreshEventData data = new RefreshEventData(context, currentIndex, newIndex);
      this.publisher.publishEvent(new RefreshEvent(this, data, data.toString()));
    ...
  }

```
	

RefreshEventListener
```
	@Override
	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof ApplicationReadyEvent) {
			handle((ApplicationReadyEvent) event);
		}
		else if (event instanceof RefreshEvent) {
			handle((RefreshEvent) event);
		}
	}

	public void handle(RefreshEvent event) {
		if (this.ready.get()) { // don't handle events before app is ready
			log.debug("Event received " + event.getEventDesc());
			Set<String> keys = this.refresh.refresh(); --> refresh is a ContextRefresher
			log.info("Refresh keys changed: " + keys);
		}
	}
```

ContextRefresher
when extract properties we see a BootstrapPropertySource, which contains a ConsulPropertySource delegate and a ConsulClient as source.
```
	public synchronized Set<String> refreshEnvironment() {
		Map<String, Object> before = extract(this.context.getEnvironment().getPropertySources());
		addConfigFilesToEnvironment();  --> This will pull config from consul
		Set<String> keys = changes(before,
				extract(this.context.getEnvironment().getPropertySources())).keySet(); --> extract the changes values from the ConsulPropertySource updated at the last step
		this.context.publishEvent(new EnvironmentChangeEvent(this.context, keys));
		return keys;
	}

```

addConfigFilesToEnvironment
Build a dummy application based on a copied current environment, to then call run() to build the application context. The PropertySourceBootstrapConfiguration will be rebuilt.
```
    StandardEnvironment environment = copyEnvironment(this.context.getEnvironment());
    SpringApplicationBuilder builder = new SpringApplicationBuilder(Empty.class)
        .bannerMode(Mode.OFF).web(WebApplicationType.NONE)
        .environment(environment);
    builder.application()
        .setListeners(Arrays.asList(new BootstrapApplicationListener(),
            new ConfigFileApplicationListener()));
    builder.run();

```

In PropertySourceBootstrapConfiguration.initialize(...) it calls ConsulPropertySourceLocator to build the ConsulPropertySource.
ConsulPropertySource then query Consul to get the latest change and rebuild the properties with the latest index.
```
	public void init() {

		Response<List<GetValue>> response = this.source.getKVValues(this.context,
				this.configProperties.getAclToken(), QueryParams.DEFAULT);

		this.initialIndex = response.getConsulIndex();

		final List<GetValue> values = response.getValue();
		ConsulConfigProperties.Format format = this.configProperties.getFormat();
		switch (format) {
		case KEY_VALUE:
			parsePropertiesInKeyValueFormat(values);
			break;
		case PROPERTIES:
		case YAML:
			parsePropertiesWithNonKeyValueFormat(values, format);
		}
```

After the dummy application is initialized, use the changed properties in the dummy environment to replace the current properties.
```
    MutablePropertySources target = this.context.getEnvironment().getPropertySources();
    for (PropertySource<?> source : environment.getPropertySources()) {
      String name = source.getName();
    if (target.contains(name)) {
      target.replace(name, source);
    }
```

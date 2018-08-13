# presto

        ##阅读PrestoServer.java

        package com.facebook.presto.server;
        public class PrestoServer implements Runnable
        {
        
            private static Announcer announcer;
            private static ServerConfig serverConfig;
            public enum DatasourceAction {
                ADD, DELETE;
            }

            public static void main(String[] args)
            {
                new PrestoServer().run();
            }

            private final SqlParserOptions sqlParserOptions;

            public PrestoServer()
            {
                this(new SqlParserOptions());
            }

            public PrestoServer(SqlParserOptions sqlParserOptions)
            {
                this.sqlParserOptions = checkNotNull(sqlParserOptions, "sqlParserOptions is null");
            }


        ##main方法中执行PrestoServer.run方法
    
            public void run()    
            {
                // 小于java 1.8
                // 不是Oracle Corporation，报错
                // 不是64位，报错
                // Linux系统时 不是amd64，报错
                // Mac OS X系统时，不是x86_64,报错
                // 不是Linux或Mac OS X，报错
                verifyJvmRequirements(); 

                Logger log = Logger.get(PrestoServer.class);

                // 定义 modules为不可变的List
                ImmutableList.Builder<Module> modules = ImmutableList.builder();
                modules.add(
                        new NodeModule(),
                        new DiscoveryModule(),
                        new HttpServerModule(),
                        new JsonModule(),
                        new JaxrsModule(true),
                        new MBeanModule(),
                        new JmxModule(),
                        new JmxHttpModule(),
                        new LogJmxModule(),
                        new TraceTokenModule(),
                        new JsonEventModule(),
                        new HttpEventModule(),
                        new EmbeddedDiscoveryModule(),
                        new ServerMainModule(sqlParserOptions));
                // add 个空值        
                modules.addAll(getAdditionalModules());
                // 定义一个引导程序 (定义log,初始化log设为true,显示Binding设为true, 复制modules)
                Bootstrap app = new Bootstrap(modules.build());

                try {
                    Injector injector = app.strictConfig().initialize();
                    serverConfig = injector.getInstance(ServerConfig.class);

                    injector.getInstance(PluginManager.class).loadPlugins();

                    injector.getInstance(CatalogManager.class).loadCatalogs();

                    announcer = injector.getInstance(Announcer.class);
                    // TODO: remove this huge hack
                    updateDatasources(
                            announcer,
                            injector.getInstance(Metadata.class),
                            injector.getInstance(ServerConfig.class),
                            injector.getInstance(NodeSchedulerConfig.class));

                    announcer.start();

                    log.info("======== SERVER STARTED ========");
                }
                catch (Throwable e) {
                    log.error(e);
                    System.exit(1);
                }
            }

            protected Iterable<? extends Module> getAdditionalModules()
            {
                return ImmutableList.of();
            }
    
          // ImmutableList.java
             public static <E> ImmutableList<E> of() {
                 return (ImmutableList<E>) EMPTY;
             }

        // announcer.start方法
        public void start() {
                Preconditions.checkState(!this.executor.isShutdown(), "Announcer has been destroyed");
                //CAS方法
                if(this.started.compareAndSet(false, true)) {
                    try {
                        this.announce().get(30L, TimeUnit.SECONDS);
                    } catch (Exception var2) {
                        ;
                    }
                }
         }
         
            /**
             *  如果当前值等于期待的值，将值设置至给定的更新值。
             * Atomically sets the value to the given updated value
             * if the current value {@code ==} the expected value.
             *
             * @param expect the expected value
             * @param update the new value
             * @return {@code true} if successful. False return indicates that
             * the actual value was not equal to the expected value.
             */
            public final boolean compareAndSet(boolean expect, boolean update) {
                int e = expect ? 1 : 0;
                int u = update ? 1 : 0;
               // CAS方法
               //如果主内存中的值是期望的e则替换成u并返回true,否则返回false。
                return unsafe.compareAndSwapInt(this, valueOffset, e, u);
            }

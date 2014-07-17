atomikos_jta
============

Using atomikos on Hibernate transaction and Hornetq transacted jmsMessageTemplate


 * xaDataSource and JtaTransaction Manager 

	
	   @Bean(destroyMethod = "close",initMethod="init")
	    public DataSource myxaDataSource() throws AtomikosSQLException {
			Properties xaProperties = new Properties();
			xaProperties.put("user", user);
			xaProperties.put("password", password);
			xaProperties.put("url", url);
			xaProperties.put("pinGlobalTxToPhysicalConnection", true);
			//xaProperties.put("com.atomikos.icatch.serial_jta_transactions", false);
			com.atomikos.jdbc.AtomikosDataSourceBean datasourceBean = new com.atomikos.jdbc.AtomikosDataSourceBean();
			datasourceBean.setXaDataSourceClassName("com.mysql.jdbc.jdbc2.optional.MysqlXADataSource");
			datasourceBean.setXaProperties(xaProperties);
			datasourceBean.setUniqueResourceName(PERSISTENCE_UNIT_NAME);
			//Disable test query right now
			datasourceBean.setTestQuery("select 1");
			datasourceBean.setMaxLifetime(3600);
			datasourceBean.setMinPoolSize(5);
			datasourceBean.setMaxPoolSize(30);
			datasourceBean.setReapTimeout(300);
			//datasourceBean.setMaintenanceInterval(maintenanceInterval);
			return datasourceBean;
	    }
	
	    @Bean
	    public LocalContainerEntityManagerFactoryBean myEntityManagerFactoryBean() throws AtomikosSQLException {
	
	        LocalContainerEntityManagerFactoryBean factoryBean = new LocalContainerEntityManagerFactoryBean();
	        factoryBean.setPersistenceUnitManager(myPersistenceManager());
	
	        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
	        factoryBean.setJpaVendorAdapter(vendorAdapter);
	
	        Properties jpaProperties = new Properties();
	        jpaProperties.put("hibernate.dialect", "org.hibernate.dialect.MySQLDialect");
	        jpaProperties.put("hibernate.show_sql", false);
	        jpaProperties.put("javax.persistence.transactionType", "jta");
	        jpaProperties.put("hibernate.transaction.factory_class", "org.hibernate.engine.transaction.internal.jta.CMTTransactionFactory");
	        jpaProperties.put("hibernate.transaction.manager_lookup_class", "com.atomikos.icatch.jta.hibernate3.TransactionManagerLookup");
	        //jpaProperties.put("hibernate.connection.release_mode", "on_close");
	        // enable\disable second level cache
	        if (!cacheEnabled) {
	        	jpaProperties.put("hibernate.cache.use_second_level_cache", false);           
	        } else {
	        	jpaProperties.put("hibernate.cache.region.factory_class", "org.hibernate.cache.ehcache.EhCacheRegionFactory");
	        	jpaProperties.put("net.sf.ehcache.configurationResourceName", getEhCacheConfigPath());
	        	jpaProperties.put("hibernate.cache.use_second_level_cache", true);
	            jpaProperties.put("hibernate.cache.use_query_cache", false);
	            jpaProperties.put("hibernate.cache.use_minimal_puts", true);
	            jpaProperties.put("hibernate.generate_statistics", false);
	            jpaProperties.put("hibernate.cache.use_structured_entries", true);
	        }
	
	        factoryBean.setJpaProperties(jpaProperties);
	
	        return factoryBean;
	    }
	    
	    @Bean
	    public DefaultPersistenceUnitManager myPersistenceManager() throws AtomikosSQLException {
	        DefaultPersistenceUnitManager dpum = new DefaultPersistenceUnitManager();
	        dpum.setDefaultDataSource(myDataSource());
	        dpum.setPackagesToScan(ENTITY_PACKAGES);
	        dpum.setDefaultPersistenceUnitName(PERSISTENCE_UNIT_NAME);
	        return dpum;
	    }
	    
	    @Bean
	    @Qualifier(PERSISTENCE_UNIT_NAME)
	    public PlatformTransactionManager transactionManager() throws SystemException {
	    	org.springframework.transaction.jta.JtaTransactionManager transactionManager = new org.springframework.transaction.jta.JtaTransactionManager();
	    	J2eeUserTransaction userTransaction = new J2eeUserTransaction();
	    	transactionManager.setTransactionManager(getAtomikosTransactionManager());
	    	transactionManager.setUserTransaction(userTransaction);
	    	userTransaction.setTransactionTimeout(30);
	    	transactionManager.setAllowCustomIsolationLevels(true);
	        return transactionManager;
	    }
	    
	    @Bean(destroyMethod = "close",initMethod="init")
	    public UserTransactionManager getAtomikosTransactionManager(){
	    	com.atomikos.icatch.jta.UserTransactionManager userTransactionManager = new UserTransactionManager();
	      	userTransactionManager.setStartupTransactionService(true);
	    	return userTransactionManager;
	    	
	    }

 
    
> * Horentq Jms xa connection factory
	
		<bean id="xaConnectionFactory" class="xx.ConnectionFactoryBean"
			init-method="init" destroy-method="close">
			<!-- The unique resource name needed for recovery by the Atomikos core. -->
			<property name="uniqueResourceName" value="XA_UNIQUE_NAME"/>
			<property name="xaConnectionFactory" ref="hornetQClient"/>
			<property name="poolSize" value="10"/>
		    <property name="username" value="name"/>
		    <property name="password" value="password"/>
			
		</bean>
		<util:constant id="QUEUE_XA_CF"
			static-field="org.hornetq.api.jms.JMSFactoryType.QUEUE_XA_CF" />
	
		<bean name="hornetQClient" class="org.hornetq.api.jms.HornetQJMSClient"
			factory-method="createConnectionFactoryWithoutHA">
			<constructor-arg index="0" ref="QUEUE_XA_CF" />
			<constructor-arg index="1" ref="transportConnectionFactory" />
		</bean>
		
		<bean id="transportConnectionFactory" class="xxxx.TransportConfigurationFactory" factory-method="createConnectors">
	        <constructor-arg value="${hornetq.hosts}" />
	    </bean>
	    <bean id="myQueue" class="org.hornetq.jms.client.HornetQQueue">
	      <constructor-arg index="0" value="MyQueue"/>
	    </bean>
	    
	    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
	        <property name="connectionFactory" ref="xaConnectionFactory" />
	        <property name="defaultDestination" ref="myQueue"/>
	        <property name="sessionTransacted" value="true"/>
	        <property name="receiveTimeout" value="10000"/> 
	    </bean>
	

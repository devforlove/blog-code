저는 spring으로 어플리케이션을 개발하면서 ```@Autowired```로 빈 주입을 사용해왔습니다. 하지만 부끄럽게도 구체적으로 '어떻게' 주입이 되는지에 대해 동작원리를 모르고 사용해왔습니다.
단순히 ```@Autowired``` 붙이면 '빈을 가져다 쓸 수 있다.', '싱글톤으로 생성된다.' 정도만 알고 사용해왔던 것 같습니다. 

그래서 spring boot 소스코드를 바탕으로 ```@Autowired```가 내부적으로 어떻게 빈을 주입하는지 알아보고자 합니다.
추가적으로 ```@Primary```, ```@Qualifier``` 어노테이션도 함께 알아보겠습니다. 


## 코드로 살펴보기 

beanFactory에서 bean을 주입하는 부분은 DefaultListableBeanFactory.doResolveDependency() 입니다. 

```java
@Nullable
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

    InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
    try {
        Object shortcut = descriptor.resolveShortcut(this);
        if (shortcut != null) {
            return shortcut;
        }

        Class<?> type = descriptor.getDependencyType();
        Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
        if (value != null) {
            if (value instanceof String strValue) {
                String resolvedValue = resolveEmbeddedValue(strValue);
                BeanDefinition bd = (beanName != null && containsBean(beanName) ?
                        getMergedBeanDefinition(beanName) : null);
                value = evaluateBeanDefinitionString(resolvedValue, bd);
            }
            TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
            try {
                return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
            }
            catch (UnsupportedOperationException ex) {
                // A custom TypeConverter which does not support TypeDescriptor resolution...
                return (descriptor.getField() != null ?
                        converter.convertIfNecessary(value, type, descriptor.getField()) :
                        converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
            }
        }

        Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
        if (multipleBeans != null) {
            return multipleBeans;
        }

        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor); //(1)
        if (matchingBeans.isEmpty()) { //(2)
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            return null; 
        }

        String autowiredBeanName;
        Object instanceCandidate;

        if (matchingBeans.size() > 1) {
            autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor); // (3)
            if (autowiredBeanName == null) {
                if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                    return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
                }
                else {
                    // In case of an optional Collection/Map, silently ignore a non-unique case:
                    // possibly it was meant to be an empty collection of multiple regular beans
                    // (before 4.3 in particular when we didn't even look for collection beans).
                    return null;
                }
            }
            instanceCandidate = matchingBeans.get(autowiredBeanName);
        }
        else {
            // We have exactly one match.
            Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
            autowiredBeanName = entry.getKey();
            instanceCandidate = entry.getValue();
        }

        if (autowiredBeanNames != null) {
            autowiredBeanNames.add(autowiredBeanName);
        }
        if (instanceCandidate instanceof Class) {
            instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
        }
        Object result = instanceCandidate;
        if (result instanceof NullBean) {
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            result = null;
        }
        if (!ClassUtils.isAssignableValue(type, result)) {
            throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
        }
        return result;
    }
    finally {
        ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
    }
}
```

(1): ```findAutowireCandidates``` 메서드는 autowired의 후보인 빈들을 반환합니다. Map의 키에는 빈 이름이 들어가고, 벨류에는 빈 인스턴스가 들어갑니다.
```findAutowireCandidates``` 메서드의 내부 코드를 확인해보면 타입이 일치하는 빈 인스턴스를 벨류로 넣어주는 것을 확인할 수 있습니다. 
아래는 ```findAutowireCandidates```의 일부입니다. 

```java
for (Map.Entry<Class<?>, Object> classObjectEntry : this.resolvableDependencies.entrySet()) {
    Class<?> autowiringType = classObjectEntry.getKey();
    if (autowiringType.isAssignableFrom(requiredType)) { // 타입이 일치하는 빈 인스턴스를 찾아서 맵에 넣어주는 것을 확인할 수 있습니다.
        Object autowiringValue = classObjectEntry.getValue();
        autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
        if (requiredType.isInstance(autowiringValue)) { 
            result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
            break;
        }
    }
}
```
(2): 주제와는 어긋나지만 중요한 부분입니다. ```@Autowired```로 빈을 주입받을 때, 일치하는 타입의 빈이 없다면 null을 주입합니다. 
만약 ```@Autowired```로 필드 주입을 한다면 런타임 시점에 NullPointerException이 발생할 것입니다. 하지만 생성자 주입을 한다면 final 접근자로 인해 스프링 컨텍스트 생성 시점에 오류가 발생합니다. 
그래서 런타임 시점에 NullPointerException이 발생하는 것을 막기 위해서 생성자 주입을 사용해서 빈을 주입합니다. 

(3): ```determineAutowireCandidate``` 로직은 일치하는 빈이 여러개일 경우 어떤 빈을 선택할지를 결정합니다. 
아래는 ```determineAutowireCandidate``` 코드입니다.

```java
@Nullable
protected String determineAutowireCandidate(Map<String, Object> candidates, DependencyDescriptor descriptor) {
    Class<?> requiredType = descriptor.getDependencyType();
    String primaryCandidate = determinePrimaryCandidate(candidates, requiredType); // @primary이면 바로 적용 
    if (primaryCandidate != null) {
        return primaryCandidate;
    }
    String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
    if (priorityCandidate != null) {
        return priorityCandidate;
    }
    // Fallback
    for (Map.Entry<String, Object> entry : candidates.entrySet()) {
        String candidateName = entry.getKey();
        Object beanInstance = entry.getValue();
        if ((beanInstance != null && this.resolvableDependencies.containsValue(beanInstance)) ||
                matchesBeanName(candidateName, descriptor.getDependencyName())) {
            return candidateName;
        }
    }
    return null;
}
```

코드에서 확인할 수 있듯이. ```@Primary``` 어노테이션이 있으면 선주입하게 됩니다. 다음으로 ```@Priority``` 어노테이션을 가진 빈이 주입됩니다. 
마지막으로 아무 어노테이션이 없을 경우는 빈 이름으로 주입됩니다. 

여기서 중요한 부분은 ```@Primary``` 어노테이션을 사용할 경우 빈이름을 무시하고 무조건 ```@Primary``` 어노테이션만 적용된다는 점입니다. 만약 누군가 아무도 모르게 ```@Primary``` 어노테이션을 적용하면 묵시적인 빈 이름을 무시하고 같은 타입의 모든 빈 주입에 
```@Primary```가 적용됩니다. 그래서 ```@Primary```는 조심히 사용해야 합니다. 

그렇다면 ```@Qualifier```는 어디에 적용되어 있을까요? getAutowireCandidateResolver().getSuggestedValue() 메서드에서 처리됩니다. 
```@Qualifier```는 matching Beans를 찾기 전에 있는지 없는지 찾아보고 Bean Injection을 합니다. 만약 일치하는 빈 이름이 없다면 오류가 발생합니다. 
```@Qualifier```가 ```@Primary``` 보다 우선 적용됨을 알 수 있습니다. 




## 직접 테스트해보자

아래 처럼 같은 타입의 빈을 등록하고 테스트하겠습니다. 

```java
@Configuration
public class FooConfig {
    @Bean
    public ObjectMapper firstMapper() {
        return new ObjectMapper();
    }

    @Bean
    public ObjectMapper secondMapper() {
        return new ObjectMapper();
    }
}

@SpringBootTest
@ActiveProfiles("test")
public class FooTest {
    @Autowired
    private ObjectMapper firstMapper;
    @Autowired
    private ObjectMapper secondMapper;

    @Test
    void test() {
        System.out.println(">>> " + firstMapper);
        System.out.println(">>> " + secondMapper);
    }
}
```
결과를 보면 서로 다른 빈이 주입되었음을 확인할 수 있습니다. 
```
>>> com.fasterxml.jackson.databind.ObjectMapper@539f0729
>>> com.fasterxml.jackson.databind.ObjectMapper@ebad9a
```

타입이 같더라도 ```determineAutowireCandidate``` 일치하는 빈이름으로 주입해주기 때문입니다. 

새로운 ```@Autowired```를 추가했습니다. 

```java
@SpringBootTest
@ActiveProfiles("test")
public class FooTest {

	@Autowired
	private ObjectMapper firstMapper;
	@Autowired
	private ObjectMapper secondMapper;
	@Autowired
	private ObjectMapper thirdMapper;

	@Test
	void test() {
		System.out.println(">>> " + firstMapper);
		System.out.println(">>> " + secondMapper);
		System.out.println(">>> " + thirdMapper);
	}
}
```
thirdMapper 라는 이름의 빈은 존재하지 않기 때문에 아래와 같은 오류가 발생합니다. firstMapper와 secondMapper 빈 중에 어떤 빈을 주입해야 할 지 모르기 때문에 발생한 오류입니다.  
```
Caused by: org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'com.fasterxml.jackson.databind.ObjectMapper' available: expected single matching bean but found 2: firstMapper,secondObjectMapper
```

만약 빈 하나를 제거한다면 위의 오류는 사라집니다. 
```java
@Configuration
public class FooConfig {
    @Bean
    public ObjectMapper firstMapper() {
        return new ObjectMapper();
    }

//    @Bean
//    public ObjectMapper secondMapper() {
//        return new ObjectMapper();
//    }
}
```

하나의 빈이 동일하게 주입됨을 확인할 수 있습니다 
```
>>> com.fasterxml.jackson.databind.ObjectMapper@4c3133a1
>>> com.fasterxml.jackson.databind.ObjectMapper@4c3133a1
>>> com.fasterxml.jackson.databind.ObjectMapper@4c3133a1
```

주석을 제거하고 두개의 빈이 등록되는 상황에서 ```@Qualifier```를 적용해보겠습니다. 

```java
@Autowired
@Qualifier("secondMapper")
private ObjectMapper thirdMapper;
```

정상적으로 빈이 주입됩니다. 
```
>>> com.fasterxml.jackson.databind.ObjectMapper@639f7054
>>> com.fasterxml.jackson.databind.ObjectMapper@26f6fbbd
>>> com.fasterxml.jackson.databind.ObjectMapper@26f6fbbd
```

이번엔 ```@Primary```를 적용해보겠습니다. 

```java
@Bean
@Primary
public ObjectMapper firstMapper() {
    return new ObjectMapper();
}
```

예상 처럼 모든 빈에 ```@Primary```가 적용됩니다. 
```
>>> com.fasterxml.jackson.databind.ObjectMapper@639f7054
>>> com.fasterxml.jackson.databind.ObjectMapper@639f7054
>>> com.fasterxml.jackson.databind.ObjectMapper@639f7054
```

```@Primary```가 적용된 상태에서 ```@Qualifier```를 적용해보겠습니다.

```java
@SpringBootTest
@ActiveProfiles("test")
public class FooTest {

	@Autowired
	private ObjectMapper firstMapper;
	
	@Autowired
    @Qualifier("secondMapper")
	private ObjectMapper secondMapper;
	
	@Test
	void test() {
		System.out.println(">>> " + firstMapper);
		System.out.println(">>> " + secondMapper);
	}
}
```

결과적으로 서로 다른 Bean이 매핑됩니다. ```@Qualifier```가 ```@Primary``` 보다 우선하기 때문입니다. 

```
>>> com.fasterxml.jackson.databind.ObjectMapper@687e561b
>>> com.fasterxml.jackson.databind.ObjectMapper@299786b1
```

## 정리
개발자는 확실함이 필수적으로 요구되는 직업이라고 생각합니다. 단 한줄 코드, 어노테이션 하나로 인해 회사에 막대한 피해를 끼칠 수 있기 때문입니다. 그렇기 때문에 '아마도 될걸', '해보니까 되더라' 라는 식의 언어는 개발자의 언어가 아니라고 생각합니다. 
그렇기에 오픈소스를 확인하는 것은 필수라고 생각합니다. 오픈소스를 확인하면 내부적으로 어떻게 동작하는지 확실하게 파악할 수 있기 때문입니다.

이번에 빈 주입 코드를 보면서 확인한 부분은 아래와 같습니다. 
- ```@Autowired```는 일치하는 빈 타입이 하나라면 해당 빈을 주입합니다. 만약 일치하는 빈이 여러개라면 빈 이름으로 주입합니다. 
- ```@Autowired```에서 일치하는 타입의 빈이 여러개인데, 이름이 일치하는 빈이 없다면 오류가 발생합니다.
- ```@Primary```가 있으면 빈 이름을 무시하고 타입기반으로 빈을 주입합니다.
- ```@Qualifier```는 ```@Primary``` 보다 우선적으로 적용됩니다. ```@Qualifier```의 빈 이름과 일치하는 빈을 주입합니다. 만약 일치하는 빈이 없으면 오류가 발생합니다. 

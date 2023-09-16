## 策略模式

### 策略模式定义

策略模式是一种对象行为型模式。策略模式定义一系列算法，将每一个算法封装起来，并让它们可以相互替换，可以让算法独立于使用它的客户而变化。

### 主要角色和UML类图

#### 主要角色

策略模式包含如下角色：

- Context: 上下文，context中定义了策略类，并且定义了一个方法，该方法在执行时会执行各具体策略中的方法。
- Strategy: 策略接口，该接口定义了一个方法，各具体的策略会实现该方法。
- ConcreteStrategy: 具体策略类，这些类都实现了Strategy策略接口中定义的方法。
- Client：客户端，在客户端中会创建Context上下文，并且指定具体的策略类，然后再执行Strategy策略接口中定义的方法。

#### UML类图

![Strategy](https://raw.githubusercontent.com/caitui/caitui.github.io/main/blog-image/Strategy.jpeg)

### Mosn访问控制模块代码分析

mosn的访问控制模块使用了策略工厂模式，访问模块定义了InheritPrincipal和InheritPermission两种策略接口，每个策略接口又有不同的类实现策略接口中定义的方法。

uml类图如下所示：

![mosn-policy](https://raw.githubusercontent.com/caitui/caitui.github.io/main/blog-image/mosn-policy.jpg)

以InheritPrincipal为例，在"filter/stream/rbac/common/principal.go"文件中首先定义了一个策略接口，该接口中有两个方法。最关键的是Match方法，不同的策略类都实现了该方法。

```go
type InheritPrincipal interface {
	isInheritPrincipal()
	// A policy matches if and only if at least one of InheritPermission.Match return true
	// AND at least one of InheritPrincipal.Match return true
	Match(cb api.StreamReceiverFilterHandler, headers api.HeaderMap) bool
}
```

定义一个根据来源IP进行规则匹配的具体策略。

```go
// Principal_SourceIp
type PrincipalSourceIp struct {
	CidrRange *net.IPNet
}

func NewPrincipalSourceIp(principal *envoy_config_rabc_v3.Principal_SourceIp) (*PrincipalSourceIp, error) {
	addressPrefix := principal.SourceIp.AddressPrefix
	prefixLen := principal.SourceIp.PrefixLen.GetValue()
	if _, ipNet, err := net.ParseCIDR(addressPrefix + "/" + strconv.Itoa(int(prefixLen))); err != nil {
		return nil, err
	} else {
		inheritPrincipal := &PrincipalSourceIp{
			CidrRange: ipNet,
		}
		return inheritPrincipal, nil
	}
}

func (principal *PrincipalSourceIp) Match(cb api.StreamReceiverFilterHandler, headers api.HeaderMap) bool {
	remoteAddr := cb.Connection().RemoteAddr()
	addr, err := net.ResolveTCPAddr(remoteAddr.Network(), remoteAddr.String())
	if err != nil {
		panic(fmt.Errorf("[PrincipalSourceIp.Match] failed to parse remote address in rbac filter, err: %v", err))
	}
	if principal.CidrRange.Contains(addr.IP) {
		return true
	} else {
		return false
	}
}
```

定义一个根据Header进行规则匹配的具体策略。

```go
// Principal_Header
type PrincipalHeader struct {
	Target      string
	Matcher     HeaderMatcher
	InvertMatch bool
}

func NewPrincipalHeader(principal *envoy_config_rabc_v3.Principal_Header) (*PrincipalHeader, error) {
	inheritPrincipal := &PrincipalHeader{}
	inheritPrincipal.Target = principal.Header.Name
	inheritPrincipal.InvertMatch = principal.Header.InvertMatch
	if headerMatcher, err := NewHeaderMatcher(principal.Header); err != nil {
		return nil, err
	} else {
		inheritPrincipal.Matcher = headerMatcher
		return inheritPrincipal, nil
	}
}

func (principal *PrincipalHeader) Match(cb api.StreamReceiverFilterHandler, headers api.HeaderMap) bool {
	targetValue, found := headerMapper(principal.Target, headers)

	// HeaderMatcherPresentMatch is a little special
	if matcher, ok := principal.Matcher.(*HeaderMatcherPresentMatch); ok {
		// HeaderMatcherPresentMatch matches if and only if header found and PresentMatch is true
		isMatch := found && matcher.PresentMatch
		return principal.InvertMatch != isMatch
	}

	// return false when targetValue is not found, except matcher is `HeaderMatcherPresentMatch`
	if !found {
		return false
	} else {
		isMatch := principal.Matcher.Equal(targetValue)
		// principal.InvertMatch xor isMatch
		return principal.InvertMatch != isMatch
	}
}
```

此外，在"filter/stream/rbac/common/permissionl.go"文件中还定义了一个策略接口InheritPermission，同样有几个类实现了该接口。

```go
type InheritPermission interface {
	isInheritPermission()
	// A policy matches if and only if at least one of InheritPermission.Match return true
	// AND at least one of InheritPrincipal.Match return true
	Match(cb api.StreamReceiverFilterHandler, headers api.HeaderMap) bool
}
```

在"filter/stream/rbac/common/policy.go"文件中，定义了上下文InheritPolicy，用于存储策略对象的引用并调用具体策略类的方法。

```go
type InheritPolicy struct {
	// The set of permissions that define a role.
	// Each permission is matched with OR semantics.
	// To match all actions for this policy, a single Permission with the `any` field set to true should be used.
	InheritPermissions []InheritPermission
	// The set of principals that are assigned/denied the role based on “action”.
	// Each principal is matched with OR semantics.
	// To match all downstreams for this policy, a single Principal with the `any` field set to true should be used.
	InheritPrincipals []InheritPrincipal
}
```

定义了Match方法来调用具体的策略类方法。

```go

// A policy matches if and only if at least one of its permissions match the action taking place
// AND at least one of its principals match the downstream.
func (inheritPolicy *InheritPolicy) Match(cb api.StreamReceiverFilterHandler, headers api.HeaderMap) bool {
	permissionMatch, principalMatch := false, false
	for _, permission := range inheritPolicy.InheritPermissions {
		if permission.Match(cb, headers) {
			permissionMatch = true
			break
		}
	}

	for _, principal := range inheritPolicy.InheritPrincipals {
		if principal.Match(cb, headers) {
			principalMatch = true
			break
		}
	}

	return permissionMatch && principalMatch
}
```

截止到这儿，核心的规则匹配方法已经完成，但是对于访问控制，匹配之后应该还有执行动作，比如通过或拒绝。mosn为此专门封装了一个引擎层，在"filter/stream/rbac/common/engine.go"文件中。为了减轻客户端维护不同策略算法的负担，使用了简单工厂来将不同策略维护在map中。

```go
type RoleBasedAccessControlEngine struct {
	// The request is allowed if and only if:
	//   * `action` is "ALLOWED" and at least one policy matches
	//   * `action` is "DENY" and none of the policies match
	// default is ALLOWED
	Action envoy_config_rabc_v3.RBAC_Action
	// Maps from policy name to policy. A match occurs when at least one policy matches the request.
	InheritPolicies map[string]*InheritPolicy
}
```


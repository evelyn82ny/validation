검증을 위해 Bean Validation annotation과 검증 대상 객체에 @Validated를 추가하면 된다.<br>

- error 가 발생될 때 FieldError, ObjectError 로 나눠지며
- 에러 발생에 대한 메시지가 출력되는 과정에도 정해진 규칙이 있으며
- 개발자가 직접 설정할 필요없이 Spring이 제공하는 Bean Validation만 쓰면 된다. 
- 
결국 개발자는 Spring이 제공하는 것을 편하게 가져다 쓰면 되지만, 조금 더 정확하게 파악하고자 공부한 내용을 작성한다.

# HashMap으로 error 처리

-해당 커밋 [c4ea2b2](https://github.com/evelyn82ny/validation/commit/c4ea2b222c5a1d225ca58c61442baf93475b3a46)

```java
Map<String, String> errors = new HashMap<>();

if(!StringUtils.hasText(item.getItemName())) {
    errors.put("itemName", "상품 이름은 필수입니다.");
}

if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() >1000000){
    errors.put("price", "가격은 1,000 ~ 1,000,000 까지 허용합니다.");
}
```
각 필드의 조건을 만족하지 못하면 에러 메시지를 HashMap에 저장하고 오류가 발생한 필드명을 key로 사용한다.<br>

```java
public String addItem(@ModelAttribute Item item, 
            RedirectAttributes redirectAttributes, Model model){
    
    if(!errors.isEmpty()) {
        model.addAttribute("errors",errors);
        return"validation/addForm";
    }
}
```
입력 필드에 대한 에러가 1개라도 존재하면 사용자가 입력 화면에 머무르는 것처럼 보이도록 하기 위해 입력 데이터와 errors를 Model에 담아 기존 view 템플릿으로 이동한다.

- model.addAttribute(errors) : 어떤 필드에서 에러가 발생했는지 알려주기 위해 **model에 error가 담긴 Map**을 저장한다.
- @ModelAttribute Item item : 에러 발생시 사용자가 입력했던 데이터를 그대로 보여주기 위해 @ModelAttribute annotation을 parameter에 작성한다.
- @ModelAttribute 작성시 **model.addAttribute("item", item)** 코드가 자동 실행되므로 객체를 직접 저장할 필요없다.


## Safe Navigation Operator

- 해당 커밋 [5993b7e](https://github.com/evelyn82ny/validation/commit/5993b7ee5e8882a0c396cd3168cd30e8a0f67fe5)

```html
<div class="field-error" th:if="${errors?.containsKey('itemName')}" 
     th:text="${errors['itemName']}">상품명 오류</di>
```
해당 필드에 해당하는 error가 포함되어 있다면 오류를 출력하는 삼항연산자가 적용된 코드이다.<br>

model 객체에 넘겨준 errors에 해당 필드 오류가 있는지 찾아보는 ```errors?.containsKey('itemName')``` 부분을 보면 **?**가 붙어있다.

해당 addForm.html 파일은 사용자가 처음 등록 화면에 들어오거나 입력 데이터를 잘못 작성했을 때 머무르는 시점에 사용된다. 처음 들어온 시점에는 당연히 에러가 발생되지 않는데 해당 코드는 errors에 에러가 존재하는지 탐색하므로 ```errors.containsKey()```로 작성시 **NullPointerException**이 발생한다. 즉, **NullPointerException** 대신 **Null**을 반환하기 위해 ? 를 추가하며 이는 SpEL에 제공하는 문법인 Safe Navigation Operator 이다.<br>


## CSS 설정

- 해당 커밋 [5993b7e](https://github.com/evelyn82ny/validation/commit/5993b7ee5e8882a0c396cd3168cd30e8a0f67fe5)

```html
.field-error {
    border-color: #dc3545;
    color: #dc3545;
}
```
오류 발생 시 오류 메시지와 해당 입력칸을 빨간색으로 설정하기 위해 style 에 색상을 추가한다.

```html
<input type="text" id="itemName" th:field="*{itemName}" 
       th:class="${errors?.containsKey('itemName')} ? 'form-control field-error' : 'form-control'"
       class="form-control" placeholder="이름을 입력하세요">
```
해당 필드 오류가 존재한다면 기본 CSS인 form-control과 style에 설정한 **field-error**가 실행되도록 한다.

# BindingResult 적용

- 해당 커밋 [b36c157](https://github.com/evelyn82ny/validation/commit/b36c157acb5527423f6c8ed7adecd7c53e534b72)

```java
@PostMapping("/add")
public String addItem(@ModelAttribute Item item, BindingResult bindingResult ...) {...}
```

BindingResult에 item 객체의 바인딩 결과를 담고 있기 때문에 BindingResult는 바인딩할 객체 바로 뒤에 작성해야한다.

## FieldError 및 ObjectError 처리

BindingResult에 error를 추가하는 방법은 다음과 같다.

- new FieldError : 필드에서 발생된 오류
- new ObjectError : 필드가 아닌 object 자체에서 발생하는 global error

### FieldError
```java
public FieldError(String objectName, String field, String defaultMessage){...}
```
```java
if (!StringUtils.hasText(item.getItemName())){
    bindingResult.addError(new FieldError("item", "itemName", "상품 이름은 필수입니다."));
}
```
필드 오류이므로 FieldError를 생성한다.

- objectName : 영향을 받는 객체명
- field : 오류가 발생한 필드명
- defaultMessage : 오류 메시지

### ObjectError

```java
public ObjectError(String objectName, String defaultMessage) {...}
```
```java
int resultPrice = item.getPrice() * item.getQuantity();
if (resultPrice < 10000) {
    bindingResult.addError(new ObjectError("globalError", "가격 * 수량의 합은 10,000원 이상이어야 합니다. 현재 값 = " + resultPrice));
}
```
글로벌 오류는 ObjectError를 생성한다.

```java
public interface BindingResult extends Errors { 
    void addError(ObjectError error);
    // more...
}
```
```java
public class FieldError extends ObjectError {...}
```
실제 addError 코드를 보면 ObjectError를 넘겨주도록 되어있는데 FieldError를 넣어도 가능한 이유는 FieldError는 ObjectError를 상속하기 때문에 가능하다.


```java
if (bindingResult.hasErrors()) {
    return "validation/addForm";
}
```
error 가 있을 경우 상품 등록화면에 사용자가 작성한 데이터가 그대로 출력되도록 model에 데이터를 담아 넘겼는데 BindingResult 를 사용하면 작성하지 않아도 된다. BindingResult를 통해 view에 자동으로 객체가 넘어가므로 model에 직접 담을 필요없다.<br>

## #fields 적용

- 해당 커밋 [fdc1d9a](https://github.com/evelyn82/validationny/commit/fdc1d9ad1858647fe5b7541d3b89d8dd30def3e9)

thymeleaf 에서 제공하는 #fields 사용 시 BindingResult 가 제공하는 검증 오류에 접근하기 위한 다양한 기능을 제공한다.

```html
<div th:if="${#fields.hasGlobalErrors()}">
```
- \#fields : BindingResult가 제공하는 검증 오류에 접근

```html
<div class="field-error" th:if="${errors?.containsKey('itemName')}" 
     th:text="${errors['itemName']}">상품명 오류</div>

<div class="field-error" th:errors="*{itemName}">상품명 오류</di>
```
- th:errors : 해당 필드에 오류가 있는 경우 tag 출력

th:errors 사용시 if문을 따로 작성할 필요없이 작성된 필드가 오류 발생되면 메시지를 출력한다.


## BindingResult 오류 발생

Integer 타입인 가격에 String을 입력하면 @ModelAttribute Binding 시점에 2가지 에러가 발생한다.

### white label error

![png](/_img/whitelabel_error_page.png){: .align-center}{: width="80%" height="80%"}

@ModelAttribute Binding 타입 오류 발생시 **BindingResult가 없으면** 400 오류와 함께 위와 같은 오류 페이지로 넘어간다. BindingResult가 없으면 오류 정보를 담을 수 없고 즉시 error로 처리된다.<br>

### Field error

![png](/_img/BindingResult_error.png){: .align-center}{: width="80%" height="80%"}

@ModelAttribute Binding 타입 오류 발생시 **BindingResult가 있으면** 해당 필드에 대한 오류를 BindingResult에 담아 Controller를 호출한다. 필드에 대한 오류이기 때문에 FieldError이며 Spring이 알아서 FieldError를 생성해 BindingResult에 담는다. 
Controller가 정상적으로 호출되기 때문에 상품 등록화면과 해당 에러 메시지를 출력한다.<br>

BindingResult에 validation error를 담는 방법은 다음과 같다.

- 위 경우과 같이 Spring이 알아서 생성해 담는 경우
- bindingResult.addError(new FieldError(...)) 작성해 직접 담는 경우
- Validator 사용 (아래 Validator 참고)


## rejectedValue 잘못된 입력도 데이터 유지

- 해당 커밋 [8faf58b](https://github.com/evelyn82ny/validation/commit/8faf58b0512fda1d57f4038c6342ba673de5257f)

타입이 일치하지 않거나 조건에 충족되지 않은 잘못된 데이터를 작성해도 사용자에게 그대로 보여주기 위한 설정을 한다.<br>

![png](/_img/data_not_retained_when_incorrectly_entered.png){: .align-center}{: width="80%" height="80%"}

- field error : 1,000원 부터 입력받기로한 가격 필드에 10원을 입력해 전송하면 가격 데이터만 제외하고 출력된다.
- binding 실패 : Integer type인 가격 필드에 String 입력하면 가격 데이터만 제외하고 출력된다.

```java
public FieldError(String objectName, String field, String defaultMessage){...}
```
FieldError를 위 생성자로 생성하면 잘못 입력된 데이터를 유지할 수 있는 방법이 없다. 숫자가 아닌 문자 입력하면 Integer type에 String을 저장할 수 없기 때문이다.

아래 FieldError의 다른 생성자로 생성 시 오류가 발생해도 입력 데이터를 저장할 수 있다. 
```java
public FieldError(String objectName, String field, @Nullable Object rejectedValue, 
                    boolean bindingFailure, @Nullable String[] codes, 
                    @Nullable Object[] arguments, @Nullable String defaultMessage) {...}
```
```java
bindingResult.addError(new FieldError("item", "price", item.getPrice(), 
                        false, null,
                        null, "가격은 1,000 ~ 1,000,000 까지 허용합니다."));
```
- objectName : 오류가 발생한 객체명
- field : 오류 필드
- rejectedValue : 사용자가 잘못 입력한 데이터
- bindingFailure : 타입 불일치 같은 binding 실패(true)인지 validation 오류(false)인지 구분
- codes : 메시지 코드 (아래 message 기능 참고)
- arguments : 메시지에서 사용하는 인자 (아래 message 기능 참고)
- defaultMessage : 기본 오류 메시지

![png](/_img/data_retained_when_incorrectly_entered.png){: .align-center}{: width="80%" height="80%"}

Binding / validation 오류 발생시 rejectedValue에 잘못 입력된 데이터를 저장하기 때문에 데이터 유지가 가능하다.

### th:field

```html
<label for="price" th:text="#{label.item.price}">가격</label>
<input type="text" id="price" th:field="*{price}"
       th:errorclass="field-error" class="form-control"
       placeholder="가격을 입력하세요">
```
사용자가 모든 필드를 제대로 입력한 경우 **th:field="\*{price}"** 에서 Item 객체의 price값을 getter로 읽어와 출력한다. 하지만 오류 발생시 th:field가 알아서 FieldError에 담긴 **rejectedValue** 값을 출력한다.

즉, Integer 타입인 가격 필드에 String을 입력하면 Spring은 아래와 같은 FieldError를 생성해 BindingResult에 담는다.
```java
new FieldError("item", "price", "잘못된 데이터(string)", true,   // Binding Error이므로 true 설정
        null, null, "Spring에서 제공하는 defaultMessage");
```
thymeleaf의 th:field가 rejectedValue에 저장된 잘못된 데이터를 출력하기 때문에 Binding 실패해도 데이터를 출력할 수 있게된다.


# 오류 메시지 관리 및 error 자동 생성

에러 발생 지점에서 에러를 생성할 때마다 오류 메시지를 작성하는 것보다 Spring이 제공하는 Message기능을 사용하고 에러 메시지 레벨을 설정해 오류 메시지를 효율적으로 관리한다. 또한, 개발자가 FieldError, ObjectError를 직접 생성하지 않고 BindingResult가 알아서 생성하도록 처리할 수 있다.<br>

## Message 기능

- 해당 커밋 [f1b2eb1](https://github.com/evelyn82ny/validation/commit/f1b2eb1a3fd03499221f54759ef3f1cdf3e5e9d1)

같은 에러 메시지를 여러 곳에서 사용할 땐 Spring이 제공하는 Message 기능을 사용하면 효율적인 관리가 가능하다. messages.properties 에 작성해도 되지만 오류만 따로 작성하는 errors.properties를 생성했다.<br>
```text
range.item.price=가격은 {0}원 ~ {1}원까지 허용합니다.
```

```메시지 코드 = 오류 메시지 내용``` 형식으로 작성한다.

errors.properties에 등록한 메시지를 Controller에 적용하기 위해 FieldError 생성자의 codes, arguments를 사용한다.

- 해당 커밋 [ff7bd8c](https://github.com/evelyn82ny/validation/commit/ff7bd8c717a3eddf28b7943083d27b958ba12dc7)

```java
bindingResult.addError(new FieldError("item", "price", item.getPrice(), false, 
  new String[]{"range.item.price"}, new Object[]{1000, 1000000} , null));
```

이전에는 defaultMessage에 오류 메시지를 직접 작성했다면 message 기능을 사용하기 위해 codes 와 arguments 를 이용한다. codes 에는 메시지 코드, arguments 에는 {0}, {1} 같은 메시지의 인자 값을 작성하는데 **둘다 배열로 작성**한다.

배열에 작성된 메시지 코드를 순서대로 매칭을 시도해 처음으로 매칭되는 메시지가 사용되며 모든 메시지 매칭을 실패하는 경우 defaultMessage 가 출력된다.

## rejectValue

- 해당 커밋 [8802cb8](https://github.com/evelyn82ny/validation/commit/8802cb802948a9eaa7678d45b6bc23c4b24a17f1)

BindingResult 를 검증해야 할 객체 바로 뒤에 작성하므로 BindingResult는 검증해야 할 객체(target)가 무엇인지 알고 있다. target 을 알고 있으니 FieldError, ObjectError 를 생성 시 objectName을 작성할 필요가 없다.

또한, 이전까지 FieldError, ObjectError 를 직접 생성했다면, BindingResult가 제공하는 **rejectValue(), reject()**를 사용해 오류가 자동 생성되도록 한다.

### FieldError -> rejectValue()

```java
void rejectValue(@Nullable String field, String errorCode,
@Nullable Object[] errorArgs, @Nullable String defaultMessage);
```
```java
bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000} , null);
```
FieldError는 rejectValue method를 사용한다.

- field : 오류 필드명
- errorCode : messageResolver를 위한 오류 코드 (아래 MessageCodesResolver 참고)
- errorArgs : 오류 메시지의 인자 값
- defaultMessage : 오류 메시지를 찾을 수 없을 때 사용하는 기본 메시지

여기서 주의할 점은 errorCode이다.

FieldError, ObjectError를 직접 생성할 때는 errors.properties에 등록된 메시지 코드를 그대로 작성했다면, BindingResult의 errorCode는 messageResolver를 위한 오류 코드이다. 위 예시를 보면 errorCode에 properties 파일에 작성된 메시지 코드를 그대로 적지 않고, 앞 일부분만 작성했는데 해당 내용은 아래 Error Level을 참고하면 된다.

### ObjectError -> reject()
```java
void reject(String errorCode, @Nullable Object[] errorArgs, @Nullable String defaultMessage);
```
```java
bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);
```

ObjectError는 reject method를 사용한다.

## MessageCodesResolver

FieldError, ObjectError 를 직접 생성 시 메세지 코드는 String[] 으로, 전달 인자값은 Object[]로 전달했다. 하지만 BindingResult의 rejectValue(), reject() 사용 시 **String 타입**의 messageResolver를 위한 오류 코드를 넘겨주면서 더 간단해졌다.

messageResolver를 위한 오류 코드가 무슨 뜻인지 알기 위해선 몇가지 개념을 알아야 한다.

일단 MessageCodesResolver는 검증 오류 코드로 메시지 코드들을 생성한다. 검증 오류 코드와 관련된 오류 메시지 코드를 모두 생성하는데 여기서 검증 오류 코드(messageResolver를 위한 오류 코드)는 BindingResult가 제공하는 rejectedValue(), reject()의 errorCode에 해당된다. 즉, 이전처럼 개발자가 관련된 오류를 일일이 작성할 필요없이 간단한 String만 작성하면 되는데 간단한 String이 무엇인지 알기 위해선 Error Level에 대해 알아야 한다.

### Error Level

```html
required = 필수 값 입니다.
```
'필수 값 입니다.'라는 에러 메시지가 여러 종류의 필드에서 필요하다면 1개의 에러 메시지를 여러 필드에서 사용해 활용성을 높인다.

```html
required.item.itemName=상품 이름은 필수입니다.
```
하지만 위와 같이 Item 객체의 itemName 필드에서만 사용되는 에러 메시지는 구체적이지만 활용성이 떨어진다. 활용성 높은 에러 메시지만 만들기엔 구체적이지 않고, 구체적인 에러 메시지를 일일이 만들기엔 너무 많아 관리하기 힘든 문제가 발생한다. 이를 위해 Error Level 이 만들어졌다.

- 해당 커밋 [166e21f](https://github.com/evelyn82ny/validation/commit/166e21f84f40f66984e440a6c3e2506f0d18c309)

```html
#Level1
required.item.itemName=상품 이름은 필수입니다.

#Level3
required.java.lang.String = 필수 문자입니다.

#Level4
required = 필수 값 입니다.
```
레벨 숫자가 작을수록 구체화된 에러 메시지이며 실제 메시지 생성 규칙이 존재한다. 잘보면 맨 앞이 **required**로 모두 동일한데 이 부분이 errorCode 이다.

객체 오류에 대한 메시지 생성 규칙은 다음과 같다.

|순서|오류 코드|예시|
|:-----:|:--------------:|:--------------:|
|1|code + "." + object name|required.item|
|2|code|required|


필드 오류에 대한 메시지 생성 규칙은 다음과 같다.

|순서|오류 코드|예시|
|:-----:|:--------------:|:--------------:|
|1|code + "." + object name + "." + field name|typeMismatch.item.itemName|
|2|code + "." + field name|typeMismatch.itemName|
|3|code + "." + field type|typeMismatch.java.lang.Sting|
|4|code|typeMismatch|


MessageCodesResolver는 level1처럼 구체적인 메시지부터 만든다. 예를 들어 itemName의 requried 검증 오류 메시지가 발생하면 다음 순서대로 메시지가 생성된다.

```text
1. required.item.itemName
2. required.itemName
3. required.java.lang.Sting
4. required
```

해당 순서로 생성된 메시지 코드를 기반으로 errors.properties 같은 MessageSource에서 메시지를 매칭한다.

![png](/_img/set_message_for_binding_error.png){: .align-center}{: width="80%" height="80%"}

개발자가 작성한 메시지 코드와 매칭에 성공하면 해당 오류 메시지가 출력된다.

![png](/_img/BindingResult_error.png){: .align-center}{: width="80%" height="80%"}

만약 MessageCodesResolver가 만든 모든 메시지에 대해 매칭이 실패한다면 Spring이 제공하는 기본 오류 메시지가 출력된다.

# Controller에서 Validation logic 분리

Validation 부분을 따로 관리하기 위해 Controller에서 Validation logic을 분리하고 Spring이 제공하는 Validator interface를 사용해 체계적인 검증 기능을 이용한다.

## Validator

- 해당 커밋 [c3ffe3c](https://github.com/evelyn82ny/validation/commit/c3ffe3ca1ffc463922ffc16f7b2cf0196a2a369e)

Item 객체를 검증하기 위한 ItemValidator 클래스를 생성하는데 Spring에서 제공하는 검증 기능인 Validator interface를 상속받는다.

```java
public interface Validator {
    boolean supports(Class<?> clazz);
    void validate(Object target, Errors errors);
}
```
```java
public class ItemValidator implements Validator { 
    @Override
    public boolean supports(Class<?> clazz) {
        return Item.class.isAssignableFrom(clazz);
    }
    // more...
}
```
- supports : 검증 대상 객체를 해당 검증기가 지원하는지 체크
    - 검증기가 많아지면 검증 대상 객체를 어떤 검증기에서 검증해야 하는지 선택해야 하기 때문
    - supports(item) 이 호출되면 Item과 동일한 객체이므로 return true (자식 객체에 속해도 true가 반환된다.)
    - isAssignableFrom을 통해 Item의 자식 객체까지 확인 가능
- validate : 검증 대상 객체를 검증해 오류 발생시 BindingResult에 오류 저장


```java
itemValidator.validate(item, bindingResult);

if (bindingResult.hasErrors()) {
  return "validation/addForm";
}

// 성공 로직
```
controller에서 검증 대상 객체와 BindingResult를 보내면 itemValidator에서 객체를 검증한다. 만약 해당 객체에 오류가 있을 경우 bindingResult에 오류가 담겨 오기 떄문에 if문으로 처리한다.

## @InitBinder, @Validated

- 해당 커밋 [e2da440](https://github.com/evelyn82ny/validation/commit/e2da440ea7fcf05bc5fea41eaedc898e883fe7c7)

### WebDataBinder

Spring은 파라미터 바인딩 역할 및 검증 기능인 **WebDataBinder**를 제공한다.

```java
@InitBinder
public void init(WebDataBinder dataBinder) {
  dataBinder.addValidators(itemValidator);
}
```
WebDataBinder에 검증기를 추가하면 해당 controller에서 추가된 검증기를 자동 적용한다. @InitBinder 작성시 해당 controller 내에서만 적용된다.

### @Validated

```java
public String addItem(@Validated @ModelAttribute Item item, BindingResult
                    bindingResult, RedirectAttributes redirectAttributes) {...}
```
controller의 각 method에서 validator를 직접 호출하는 대신 검증받고자 하는 객체 앞에 @Validated 추가하면 해당 method가 실행될 때 마다 위에 작성한 init method가 실행되면서 WebDataBinder가 생성된다.

하지만 Spring을 사용하면 controller에 객체에 대한 검증기를 주입하지 않아도 되고, WebDataBinder를 생성하는 위 init method도 작성하지 않아도 된다. Spring이 알아서 다 처리해주는데 아래 Bean Validation 설명에 해당 내용을 작성했다.

- @Validated : 스프링 전용 검증 annotation
- @Valid : 자바 표준 검증 annotation

@Valid 와 @Validated의 기능은 비슷하지만, @Validated에는 상황에 따라 다른 검증 방식을 적용하기 위한 groups 기능이 포함되어 있다. 하지만 잘 사용되지 않고 폼 객체를 분리시켜 해결한다.(아래 참고)

# Bean Validation

Bean Validation을 사용하면 필드에 대한 검증 로직을 직접 작성할 필요가 없다.

```java
if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) { 
    errors.rejectValue("price", "range", new Object[]{1000, 1000000} , null);
}
```
price 필드에 대한 조건을 설정하고 만족하지 못하면 rejectValue method로 에러를 생성했다. 하지만 아래와 같이 Bean Validation을 작성하면 검증 로직을 일일히 작성할 필요없이 에러가 자동 생성된다.

```java
@NotNull  
@Range(min = 1000, max = 1000000)
private Integer price;
```
이렇게 검증 로직을 모든 곳에 적용할 수 있게 표준화 한 것이 Bean Validation이다.

JPA의 대표적인 구현체로 Hibernate가 있다. Bean Validation도 JPA와 마찬가지로 Interface 모음이며 Bean Validation의 대표적인 구현체로 Hibernate Validator가 있다. @NotBlank, @NotNull은 javax.validation에서 제공되지만 @Range 같은 기능은 org.hibernate.validator에서 제공된다.

## LocalValidatorFactoryBean

- 해당 커밋 [8bd439a](https://github.com/evelyn82ny/validation/commit/8bd439a35f7ecda82c6722a0ed12b3ebdccc4b73)

```java
ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
validator validator = factory.getValidator();
```
ValidatorFactory를 통해 Bean Validation를 직접 생성할 수 있다.

```text
implementation 'org.springframework.boot:spring-boot-starter-validation'
```
Bean Validation을 사용하려면 의존관계를 추가해야 하는데 의존 관계 추가 시 Spring이 자동으로 Bean Validatpr으로 인지하기 때문에 ValidatorFactory를 직접 작성하지 않는다.

Spring Boot 2.0.5 버전 이상 사용 시 javax.validation 설정을 위한 클래스인 LocalValidationFactoryBean이 자동으로 등록된다. LocalValidationFactoryBean은 @Valid 나 @Validated가 붙은 객체에 대해 Bean Validation annotation을 보고 검증한다. 즉, LocalValidationFactoryBean이 알아서 다 처리해주니 method마다 Validator를 호출할 필요가 없다.

## @ModelAttribute 검증 순서

Binding에 성공한 필드만 Bean Validation이 적용된다.

![png](/_img/set_message_for_binding_error.png){: .align-center}{: width="80%" height="80%"}

Integer type인 가격에 String을 입력하면 typeMismatch로 FieldError가 생성되며 해당 필드에 대한 **validation 이 진행되지 않고 끝난다.**

![png](/_img/data_retained_when_incorrectly_entered.png){: .align-center}{: width="80%" height="80%"}

Binding에 성공한 경우 Validation이 처리되므로 검증 오류 메시지가 출력된다.

@ModelAttribute 모든 필드 type 변환을 시도한다.(Binding)
- Binding에 실패한 필드는 typeMismatch로 FieldError를 생성
- Binding에 성공한 필드는 Bean Validation 적용

주의할 점은 1개의 필드가 Binding에 실패했다고 해당 객체가 모두 error 처리되는게 아니다. @ModelAttribute는 필드 단위로 처리되므로 특정 필드에 typeMismatch가 발생해도 나머지 필드는 정상 처리한다.

이 부분이 API JSON 요청과 다르게 처리된다. @ModelAttribute는 필드 단위로 처리되지만, API JSON 요청인 @RequestBody는 객체 단위로 처리되기 때문에 1개라도 Binding에 실패하면 해당 객체는 검증이 진행되지 않는다.

## error code

Bean Validation은 오류 메시지를 제공하는데 message 기능을 이용해 직접 작성가능하다.

### FieldError
FieldError에 대한 오류 코드는 annotation 이름으로 등록되고 MessageCodeResolver를 통해 메시지 코드가 다음과 같은 순서로 생성된다.

@NotBlank annotation에 대한 메시지 코드

```
1. NotBlank.item.itemName
2. NotBlank.itemName 
3. NotBlank.java.lang.String 
4. NotBlank
```

### ObjectError

- 해당 커밋 [b20a79a](https://github.com/evelyn82ny/validation/commit/b20a79ab09fdf315104e6487fe63f7ed36cabb3e)

```java
@ScriptAssert(lang = "javascript", script = "_this.price * _this.quantity >= 10000"
                message = "오류 메시지 내용")
```

ObjectError는 @ScriptAssert를 사용해서 처리 가능하지만 제약이 많아 잘 사용되지 않는다. 그러므로 ObjectError는 직접 자바로 작성한다.
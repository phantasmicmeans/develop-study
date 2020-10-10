## custom annotation으로 validation / hibernate 디버깅

- List<@WhiteSpace String> tags
- tags의 element 들은 모두 공백 포함 금지 스펙
- custom annotation으로 element validation check

### AS-IS
디버깅 결과 리스트를 2번씩 순회하며 check 하는 현상 발견


```java
package com.naver.spubs.contents.model.contents;

import com.fasterxml.jackson.annotation.JsonFormat;
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;

import com.naver.spubs.contents.annotation.tags.Tags;
import com.naver.spubs.contents.annotation.whitespace.WhiteSpace;
import com.naver.spubs.contents.config.jackson.CustomLocalDateDeSerializer;
import com.naver.spubs.contents.config.jackson.CustomLocalDateSerializer;

import com.naver.spubs.contents.model.newsletter.Newsletter;
import com.naver.spubs.contents.model.type.PublicType;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

import javax.validation.constraints.NotBlank;

import java.time.LocalDateTime;
import java.util.List;

@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class Contents {
    @JsonProperty("_id")
    private String _id;
    @NotBlank
    private String cpId;
    private String cpName;
    @NotBlank
    private String naverId;
    private String title;
    private String author;
    private String seJson;
    private String html;
    private String categoryId;
    private String categoryName;

    private List<@WhiteSpace String> tags;
    private String imageList;
    private String videoList;
    private String audioList;

    @JsonProperty("isDeleted")
    private boolean isDeleted;
    @JsonProperty("isPublished")
    private boolean isPublished;
    @JsonProperty("isTemp")
    private boolean isTemp;

    /**
     * ALL: 전체, MEMBER: 멤버십 회원, PRIVATE: 비공개
     */
    private PublicType publicType;

    /**
     * 컨텐츠를 발행 전 임시 저장 시간
     */
    @JsonSerialize(using = CustomLocalDateSerializer.class)
    @JsonDeserialize(using = CustomLocalDateDeSerializer.class)
    private LocalDateTime tempDatetime;

    /**
     * 컨텐츠를 발행할 시간
     */
    @JsonSerialize(using = CustomLocalDateSerializer.class)
    @JsonDeserialize(using = CustomLocalDateDeSerializer.class)
    private LocalDateTime reservedPublishDatetime;

    /**
     * 컨텐츠가 발행된 시간
     */
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss", timezone = "Asia/Seoul")
    private LocalDateTime publishDatetime;

    /**
     * 컨텐츠가 수정된 시간, 발행 이후 setting
     */
    @JsonSerialize(using = CustomLocalDateSerializer.class)
    @JsonDeserialize(using = CustomLocalDateDeSerializer.class)
    private LocalDateTime modifyDatetime;

    /**
     * 컨텐츠가 최초 등록된 시간 - 발행 버튼 클릭시 1회 세팅
     */
    @JsonSerialize(using = CustomLocalDateSerializer.class)
    @JsonDeserialize(using = CustomLocalDateDeSerializer.class)
    private LocalDateTime registerDatetime;

    private Newsletter newsletter;

    @JsonProperty("_createDatetime")
    private String _createDateTime;

    @JsonProperty("_modifyDatetime")
    private String _modifyDatetime;

    @JsonProperty("isDeleted")
    public boolean isDeleted() {
        return isDeleted;
    }

    @JsonProperty("isPublished")
    public boolean isPublished() {
        return isPublished;
    }

    @JsonProperty("isTemp")
    public boolean isTemp() {
        return isTemp;
    }
}

```

ValidatorImpl.java

```java
	private <U> boolean validateConstraintsForSingleDefaultGroupElement(BaseBeanValidationContext<?> validationContext, ValueContext<U, Object> valueContext, final Map<Class<?>, Class<?>> validatedInterfaces,
			Class<? super U> clazz, Set<MetaConstraint<?>> metaConstraints, Group defaultSequenceMember) {
		boolean validationSuccessful = true;

		valueContext.setCurrentGroup( defaultSequenceMember.getDefiningClass() );

		for ( MetaConstraint<?> metaConstraint : metaConstraints ) {
			// HV-466, an interface implemented more than one time in the hierarchy has to be validated only one
			// time. An interface can define more than one constraint, we have to check the class we are validating.
			final Class<?> declaringClass = metaConstraint.getLocation().getDeclaringClass();
			if ( declaringClass.isInterface() ) {
				Class<?> validatedForClass = validatedInterfaces.get( declaringClass );
				if ( validatedForClass != null && !validatedForClass.equals( clazz ) ) {
					continue;
				}
				validatedInterfaces.put( declaringClass, clazz );
			}

			boolean tmp = validateMetaConstraint( validationContext, valueContext, valueContext.getCurrentBean(), metaConstraint );
			if ( shouldFailFast( validationContext ) ) {
				return false;
			}

			validationSuccessful = validationSuccessful && tmp;
		}
		return validationSuccessful;
	}
```

## 디버깅 원인
```java
for ( MetaConstraint<?> metaConstraint : metaConstraints ) {
  ...
}
```

validation 항목들은 모두 metaConstraint에 포함되어 동작하는데.. 이상한 점을 발견

List<@WhiteSpace String> tags 검증시 아래 2개의 항목이 들어가있음. 따라서 2번 검증하고 있던 것.. 
  
- GetterConstraintLocation
- FieldConstraintLocation 


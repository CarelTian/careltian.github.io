# PostgreSQL


## PostgreSQL简介

PostgreSQL 是一款高级企业级开源关系数据库，支持 SQL（关系型）和 JSON（非关系型）查询。它是一个高度稳定的数据库管理系统。PostgreSQL 可用作很多 Web、移动、地理空间和分析应用程序的主要数据存储或数据仓库。

MySQL 和 PostgreSQL 都是热门的开源关系数据库。从传统意义上说，我们认为 MySQL 比较易用且速度快，而 PostgreSQL 的功能比较丰富，并且与 Oracle 等商业数据库的兼容性更好。

仓库：[https://github.com/CarelTian/database-Impl](https://github.com/CarelTian/database-Impl)

## 自定义类型

相比较于Mysql，postgresql可以有自定义类型，更灵活的扩展函数机制。下面的例子中将展示在数据库中新建一个地址类型。并且支持地址类型的索引，自定义比较，格式化输出。下图展示的是BNF	定义格式。一个地址应该由4 部分组成: DetailedUnitRoad, Suburb, State and Postcode，例如
U19/36 Queen Ave, Southgate, AR 7279

{{&lt; image src=&#34;https://raw.githubusercontent.com/CarelTian/MyStatic/master/images/postsql-1.png&#34; alt=&#34;图一&#34; width=&#34;400&#34;&gt;}}



由于地址是不定长的，在postadd.source中定义VARIABLE，输入输出函数，对齐方式为int4，这个参数可从官方文档的建议中找到。

```sql

CREATE TYPE postaddress (
   internallength = VARIABLE,
   input = postaddress_in,
   output = postaddress_out,
   alignment = int4
  -- storage = extended  天坑，加了这东西改bug 3个小时
);

```

下面是结构体定义，一个tuple字段都保存在data中，我们在结构体定义每个字段长度用于分隔。

```c
typedef struct PostAddress
{
	int32 vl_len_;
	int32 unit_len;
	int32 street_len;
	int32 suburb_len;
	int32 state_len;
	int32 postcode_len;
	int32 original_len;
	char  data[FLEXIBLE_ARRAY_MEMBER];
}PostAddress;

```





下面的代码是读取和序列号地址，这里使用了按照规则严格匹配的方法，代码冗长较难维护，但是好在通过了全部测试。还有一种方法是使用正则表达式来解析字符串，代码量和可读性都会好很多。

```c
PG_FUNCTION_INFO_V1(postaddress_in);

Datum
postaddress_in(PG_FUNCTION_ARGS)
{
	char  *str = PG_GETARG_CSTRING(0);
	PostAddress *result;
	int32 struct_size;
	char *unit = &#34;&#34;,*street=&#34;&#34;,*suburb=&#34;&#34;,*state=&#34;&#34;,*postcode=&#34;&#34;,*original;
	char *t=NULL;
	int unit_len, street_len,suburb_len,state_len,postcode_len,original_len;
	int len = strlen(str);
	char *value = (char *) palloc(len&#43;1);  
    strcpy(value, str);
	value[len]=&#39;\0&#39;;
	original=value;

	t = strtok(str, &#34;,&#34;);
	if(t==NULL ){
		ereport(ERROR,errmsg(&#34;invalid input syntax for type PostAddress: \&#34;%s\&#34;&#34;,original));
	}
	char *slash = strchr(t, &#39;/&#39;);
	if(slash==NULL){
		if(!check_street(t)) 
		ereport(ERROR,errmsg(&#34;invalid input syntax for type PostAddress: \&#34;%s\&#34;&#34;,original));
		t=to_lower(t);
		street=t;
	}else{
		*slash=&#39;\0&#39;;
		if(!check_unit(t)|| !check_street(slash&#43;1))
		ereport(ERROR,errmsg(&#34;invalid input syntax for type PostAddress: \&#34;%s\&#34;&#34;,original));
		char * s=slash&#43;1;
		t=to_lower(t);
		s=to_lower(s);
		unit=t;
		street=s;
	}
	t = strtok(NULL, &#34;,&#34;);
	if(t==NULL){
		ereport(ERROR,errmsg(&#34;invalid input syntax for type PostAddress: \&#34;%s\&#34;&#34;,original));
	}

	if(!check_suburb(t)){
		ereport(ERROR,errmsg(&#34;invalid input syntax for type PostAddress: \&#34;%s\&#34;&#34;,original));
	}
	t=to_lower(t&#43;1);  //skip space
	suburb=t;
	t = strtok(NULL, &#34;&gt;&#34;);
	if(t==NULL || t[0]!=&#39; &#39;){
		ereport(ERROR,errmsg(&#34;invalid input syntax for type PostAddress: \&#34;%s\&#34;&#34;,original));
	}
	t=t&#43;1; //skip space
	slash=strchr(t,&#39; &#39;);   
	if(slash==NULL){
		ereport(ERROR,errmsg(&#34;invalid input syntax for type PostAddress: \&#34;%s\&#34;&#34;,original));
	}else{
		*slash=&#39;\0&#39;;
		if(!check_state(t) || !check_postcode(slash&#43;1))
		ereport(ERROR,errmsg(&#34;invalid input syntax for type PostAddress: \&#34;%s\&#34;&#34;,original));
		char * s=slash&#43;1;
		t=to_lower(t);
		s=to_lower(s);
		state=t;
		postcode=slash&#43;1;
	}
	unit_len = strlen(unit);
	street_len=strlen(street);
	suburb_len=strlen(suburb);
	state_len=strlen(state);
	postcode_len=strlen(postcode);
	original_len=strlen(original);
	struct_size = (int32)(offsetof(PostAddress, data) &#43; unit_len&#43;street_len&#43;suburb_len&#43;state_len&#43;postcode_len&#43;original_len);
	result = (PostAddress *) palloc(struct_size);
	SET_VARSIZE(result, struct_size); 
	result-&gt;unit_len = unit_len;
	result-&gt;street_len   = street_len;
    result-&gt;suburb_len   = suburb_len;
    result-&gt;state_len    = state_len;
    result-&gt;postcode_len = postcode_len;
	result-&gt;original_len=original_len;
	int offset = 0;

    memcpy(result-&gt;data &#43; offset, unit, unit_len);
    offset &#43;= unit_len;

    memcpy(result-&gt;data &#43; offset, street, street_len);
    offset &#43;= street_len;

    memcpy(result-&gt;data &#43; offset, suburb, suburb_len);
    offset &#43;= suburb_len;

    memcpy(result-&gt;data &#43; offset, state, state_len);
    offset &#43;= state_len;
	
    memcpy(result-&gt;data &#43; offset, postcode, postcode_len);
    offset &#43;= postcode_len;
    memcpy(result-&gt;data &#43; offset, original, original_len);
    offset &#43;= postcode_len;

	PG_RETURN_POINTER(result);
}
```

还实现地址排序比较，索引，获取细分字段（如街道，州，邮编）可参考仓库代码。

## 多属性线性哈希文件

这个部分与postgresql不相关，只使用C来实现这个功能。

### 多属性哈希的代码

```c {data-open=true}

Bits tupleHash(Reln r, Tuple t)
{
	char buf[MAXBITS&#43;5];  //*** for debug
	Count nvals = nattrs(r);
    char **vals = malloc(nvals*sizeof(char *));
    assert(vals != NULL);
    tupleVals(t, vals);
	Bits *hash = malloc(nvals * sizeof(Bits));
	Bits comphash =0;
	for (int i = 0; i &lt; nvals; i&#43;&#43;) {
        hash[i] = hash_any((unsigned char *)vals[i], strlen(vals[i]));
        bitsString(hash[i], buf);
      //  printf(&#34;hash(%s) = %s\n&#34;, vals[i], buf);
    }
	ChVecItem *cv=  chvec(r);
	for(int i=0;i&lt;MAXCHVEC;i&#43;&#43;){
		int index=cv[i].att;
		int pos=cv[i].bit;
		Bits bit= (hash[index] &gt;&gt; pos) &amp; 1;
		comphash |=(bit &lt;&lt;i);
	}
	bitsString(comphash, buf);
	//printf(&#34;compHash: %s\n&#34;,buf);
	freeVals(vals,nvals);
	return comphash;
}
```

### 查询（选择和投影）

主要是模拟PostgreSQL的查询方式，数据库输入的每条记录对应于一条tuple，多个tuple存储在page中。每插入一个tuple，取低位的哈希值作为桶的序号插入到对应的page。所以每当查询时，就可以计算查询字符串的哈希来确定对应的桶。对于使用模糊查询，相对应的哈希位使用特殊处理的方式，即扫描所有未知哈希位的桶号。由于tuple在page中是无序连续存储的，所以并不知道具体存在page的位置编号，因此可以全部遍历所有tuple，比较是否与查询字符串匹配。投影就比较简单，知道了数据类型的结构，将存储的字符串转化为字符数组就很容易处理。

### 线性哈希

除了普通的page，还有溢出页。如果低位哈希值总是停留在少数的几个桶，那么溢出页就会越来越长，导致查询的效率降低，所以引入了分裂的概念。定义page的容量 c = floor(B/R) ≈ 1024/(10*n)，如果n=3, 那么每插入34个记录，就要对桶进行分裂操作，这里需要使用分裂指针表示即将分裂的桶。当sp = 2^d 时，d&#43;1，sp置0。使用非常巧妙的位扩容机制来扩大深度。发生扩容时，sp指向的桶及其溢出页清空，其值保存在临时tuple数组中，然后创建一个新页，也是根据低位哈希将tuple分配在这两个桶中。以此反复，page保存的值会更加分散，能够更快的查询。


---

> 作者: ZergFlood  
> URL: https://careltian.github.io/posts/doc/postgresql/  


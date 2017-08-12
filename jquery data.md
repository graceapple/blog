# jquery.data
## data的作用

为dom对象（也可以只普通js对象）添加自定义属性，使用js对象，扩充dom对象上 原先的数据存储。

通过$(element).data(key,value)为element绑定数据，$(element).data(key)获取绑定的数据。
    
## 需要解决的问题
1. 兼容html5 data-*属性
2. js对象与dom对象的循环引用

    js对象在其生命周期结束后，js的回收机制是可以将其内存释放的。但如果这个js对象还被dom对象引用的话，只要这个dom元素一直存在，就 一直无法释放内存。由于IE的dom，bom对象都是com对象，利用引用计数进行垃圾回收，如果出现dom对象与js对象循环引用，即使移除dom元素也无法释放内存，只能在需要释放内存时，手动断开js对象与dom对象之间的关系。这显然是反人类的。而jquery.data()方法，允许向dom对象绑定任意数据，避免循环引用。
3. 独立的数据存储

    既然用户数据由jquery管理，jquery.data在什么地方为每个element分配数据存储，又如何保证各个element的用户数据互不干扰，并且数据不容易被用户或其他js模块误操作。

## 大体思路

大体思路是，对于内部数据和用户数据，在dom节点（或普通js对象）上分别绑定一个对象作为存储（key用一串随机数参与生成的令牌，防止被js代码误操作）, 由jquery内部对象dataPriv和dataUser来管理，这两个对象都是jquery内部类Data类的实例。删除数据或清除dom元素时，先解除dom节点与cache数据之间的引用关系（普通js对象删除数据，直接delete）。
    
## 源码解读

1. Data
    
    Data类有一个私有属性uid，一个对象属性this.expando，prototype方法：cache，set， get， access， remove， hasData。
    
- Data.uid
```
//作为令牌expando的一部分，随着创建实例自增
Data.uid = 1;

```
- expando
```
//由jquery.expando和uid生成唯一的字符串，作为元素节点的cache的key
function Data() {
	this.expando = jQuery.expando + Data.uid++;
}

```
其中jquery.expando组成如下：
```
	// Unique for each copy of jQuery on the page
	expando: "jQuery" + ( version + Math.random() ).replace( /\D/g, "" ),
```
- cache
```
	cache: function( owner ) {

		//获取owner上的cache
		var value = owner[ this.expando ];

		//如果cache不存在，则创建一个
		if ( !value ) {
			value = {};

			//判断owner是否支持绑定数据
			if ( acceptData( owner ) ) {

				// dom元素，则直接为this.expando赋值value
				if ( owner.nodeType ) {
					owner[ this.expando ] = value;

				} else {
				    //普通js对象，this.expando属性值为value，并且要设为不可枚举，为了不被for...in枚举，defineProperty不显示配置，默认enumerable属性就是false
				    //为了将来可以delete cache[this.expando]，configurable要设为true
					Object.defineProperty( owner, this.expando, {
						value: value,
						configurable: true
					} );
				}
			}
		}

		return value;
	},

```
acceptData判断标准如下：
```
return function( owner ) {

	//只支持元素节点，document节点，和普通js对象
	return owner.nodeType === 1 || owner.nodeType === 9 || !( +owner.nodeType );
};
```
ps： 源码注释称，现代浏览器是可以为非元素节点（包括注释节点，文本节点）绑定自定义数据的，但不应该这么做
- set
```
    set: function( owner, data, value ) {
        //获取已有的cache数据，再为其添加新属性
		var prop,
			cache = this.cache( owner );

		// [ owner, key, value ] 三个命名参数都传进来的情况
		// 如果data是字符串，以data为key，为兼容HTML5 data-***，key转为驼峰式
		if ( typeof data === "string" ) {
			cache[ jQuery.camelCase( data ) ] = value;

		// 只传了owner和data，如果data是一个object，遍历它，添加其每个属性到cache上
		} else {

			//手动将data object中的所有属性赋值给cache
			for ( prop in data ) {
				cache[ jQuery.camelCase( prop ) ] = data[ prop ];
			}
		}
		return cache;
	},
```
 - get
```
    get: function( owner, key ) {
        //只传了owner，就是获取全部cache数据，如果也传了key，则返回cache[key]
		return key === undefined ?
			this.cache( owner ) :

			//如果cache存在，则返回cache[key], key需要转为驼峰式
			owner[ this.expando ] && owner[ this.expando ][ jQuery.camelCase( key ) ];
	},
```
 - access
```
    access: function( owner, key, value ) {
		// 处理get，set两种情况:
		// get: 1. 没有指定key，取整个cache；
		//      2. key是字符串，没有指定value，取cache中的key属性
		if ( key === undefined ||
				( ( key && typeof key === "string" ) && value === undefined ) ) {

			return this.get( owner, key );
		}
		// set: 1. 只传了key，key不是字符串，遍历key，添加属性到cache上
		//      2. key 和value都传了，为owner添加属性key
		this.set( owner, key, value );

		// 由于set有两种情况，返回cache添加的属性值
		//1. 指定value，则cache添加的属性值为value
		//2. 没有指定value，则cache实际新增的部分是key
		return value !== undefined ? value : key;
	},
```
 - remove
```
    remove: function( owner, key ) {
        //先获取已存在的cache
		var i,
			cache = owner[ this.expando ];
        // 如果cache不存在，就不用再删除了，直接返回
		if ( cache === undefined ) {
			return;
		}
        // 指定key，则删除cache的属性key
        //没有指定key（只传了owner，key为undefined），删除整个cache
        // 指定了key的情况
		if ( key !== undefined ) {

			// 如果key是数组，支持批量删除
			if ( Array.isArray( key ) ) {

				// key转为驼峰式
				key = key.map( jQuery.camelCase );
			} else {
			    // 对于key的操作，都要先将key转为驼峰式
				key = jQuery.camelCase( key );

				// 如果key已存在于cache中，直接使用这个key，
				//否则去掉key中的空格字符
				//最终形成一个只包含key的数组
				key = key in cache ?
					[ key ] :
					( key.match( rnothtmlwhite ) || [] );
			}

			i = key.length;

			while ( i-- ) {
				delete cache[ key[ i ] ];
			}
		}

		// 如果cache中已经没有数据了，就删除整个cache
		if ( key === undefined || jQuery.isEmptyObject( cache ) ) {

			// 由于webkit和blink在删除dom元素属性上的性能问题，
			// 不直接删除这些属性，而是使用undefined赋值cache，解除cache和js对象之间的关系
			if ( owner.nodeType ) {
				owner[ this.expando ] = undefined;
			} else {
			    // 对于普通js对象，直接使用delete删除整个cache
				delete owner[ this.expando ];
			}
		}
	},
```
PS： 解除js对象和dom元素之间的引用，赋值undefined，并不是为了循环引用问题，IE9以上dom节点不再作为com对象实现了。不直接删除属性，主要还是性能上的考虑。
 - hasData
```
    //判断owner上是否有cache，有则返回true，无则返回false
    hasData: function( owner ) {
		var cache = owner[ this.expando ];
		return cache !== undefined && !jQuery.isEmptyObject( cache );
	}
```
 2. jquery.data

这是暴露给用户的接口。

- jquery构造函数提供了几个方法: $.hasData,$.data, $.removeData, $._data, $._removeData。
```
jQuery.extend( {
    // 判断elm是否有绑定了数据，会检查用户数据和内部数据
	hasData: function( elem ) {
		return dataUser.hasData( elem ) || dataPriv.hasData( elem );
	},
    // 通过data接口绑定和查询用户数据
	data: function( elem, name, data ) {
		return dataUser.access( elem, name, data );
	},
    // 删除用户数据或用户数据中的某个属性
	removeData: function( elem, name ) {
		dataUser.remove( elem, name );
	},

	//绑定和查询内部数据
	//PS: _data, _removeData方法在内部都已经用dataPriv.access, dataPriv.removeData替代了
	//后面将deprecate这两个方法
	_data: function( elem, name, data ) {
		return dataPriv.access( elem, name, data );
	},
    //删除内部数据或内部数据的某个属性
	_removeData: function( elem, name ) {
		dataPriv.remove( elem, name );
	}
} );

```

- jquery对象只提供了 data和removeData两个方法。
```
jQuery.fn.extend( {
	data: function( key, value ) {
	    //$('selector').data(key, value), this指向elements集合
	    //elem为集合中第一个元素，elem存在，则获取attrs，elem.attributes返回一个包含attribute node的数组
		var i, name, data,
			elem = this[ 0 ],
			attrs = elem && elem.attributes;

		//$('selector').data()返回整个用户数据
		if ( key === undefined ) {
			if ( this.length ) {
			    //如果this长度不为0，则获取第一个元素的全部用户数据
				data = dataUser.get( elem );
                //如果是dom元素，且dataUser中并未包含html5的data-***数据
				if ( elem.nodeType === 1 && !dataPriv.get( elem, "hasDataAttrs" ) ) {
					i = attrs.length;
					while ( i-- ) {

						// 由于IE11中的attr可以是null，所以要先判断一下
						if ( attrs[ i ] ) {
							name = attrs[ i ].name;
							// 取data-之后的部分先转为驼峰式，以此为key，查html5的data-*属性，查询结果添加到dataUser中
							if ( name.indexOf( "data-" ) === 0 ) {
								name = jQuery.camelCase( name.slice( 5 ) );
								dataAttr( elem, name, data[ name ] );
							}
						}
					}
					//设置hasDataAttrs标志位，标识dataUser中已包含html5 data-*数据
					dataPriv.set( elem, "hasDataAttrs", true );
				}
			}

			return data;
		}

		// key为object，为dataUser绑定多个属性
		if ( typeof key === "object" ) {
			return this.each( function() {
				dataUser.set( this, key );
			} );
		}
        //其余情况：若key，value都有值，则对dataUser绑定数据后，返回链式，若只有key，没有value，则返回key属性对应的值
		return access( this, function( value ) {
			var data;

			// The calling jQuery object (element matches) is not empty
			// (and therefore has an element appears at this[ 0 ]) and the
			// `value` parameter was not undefined. An empty jQuery object
			// will result in `undefined` for elem = this[ 0 ] which will
			// throw an exception if an attempt to read a data cache is made.
			//get key属性
			if ( elem && value === undefined ) {

				// get 属性key对象的值，在dataUser中，key是驼峰式的
				data = dataUser.get( elem, key );
				if ( data !== undefined ) {
					return data;
				}

				// 如果dataUser中没有找到，继续在html5 data-*属性中查询
				data = dataAttr( elem, key );
				if ( data !== undefined ) {
					return data;
				}

				// 都找不到则返回
				return;
			}

			// 绑定新键值对 key value
			this.each( function() {

				// 存储在dataUser中的属性key必须是驼峰式
				dataUser.set( this, key, value );
			} );
		}, null, value, arguments.length > 1, null, true );
	},
    // 遍历所有元素删除属性key
	removeData: function( key ) {
		return this.each( function() {
			dataUser.remove( this, key );
		} );
	}
} );
```
 - dataAttr： 查询html5 data-*属性
```
function dataAttr( elem, key, data ) {
	var name;

	// 在内部缓存dataUser中没有找到数据，只能再到html5 data-*中发掘了，只支持元素节点
	if ( data === undefined && elem.nodeType === 1 ) {
	    //再把key从驼峰式转成data-*-*分隔线式
		name = "data-" + key.replace( rmultiDash, "-$&" ).toLowerCase();
	    //尝试获取attribute数据
		data = elem.getAttribute( name );

		if ( typeof data === "string" ) {
			try {
			    //返回正确的数据类型
				data = getData( data );
			} catch ( e ) {}

			// html5 data-*数据保存进dataUser中(属性key不包含data-前缀，在set中 会转为驼峰式),避免以后再查dom节点的attributes了
			dataUser.set( elem, key, data );
		} else {
		    //否则数据未定义
			data = undefined;
		}
	}
	return data;
}
```
 - getData： 把字符串化的数据还原为基础类型
```
function getData( data ) {
    //如果是字符串"true"， 则返回boolean值true
	if ( data === "true" ) {
		return true;
	}
    //如果是字符串"false", 则返回boolean值false
	if ( data === "false" ) {
		return false;
	}
    //如果是字符串"null", 则返回null
	if ( data === "null" ) {
		return null;
	}

	// +data把字符串转为数字，+ ""把数字再转为字符串，与原先字符串比较，若相等，则表示是数字，转成数字返回
	if ( data === +data + "" ) {
		return +data;
	}
    // 测试是否包含{}, 是则尝试转为对象
	if ( rbrace.test( data ) ) {
		return JSON.parse( data );
	}
    //上述情况都不是，则返回原本字符串
	return data;
}
```
 - access：根据参数情况get/set数据，并提供链式调用
```
// 对elems集合进行get/set操作提供多样化的接口， get返回查询数据，set则提供链式调用。
// 如果value是函数的话，可以执行该函数得到values
var access = function( elems, fn, key, value, chainable, emptyGet, raw ) {
    //如果key==null，表示数据批量操作
	var i = 0,
		len = elems.length,
		bulk = key == null;

	// 如果key是object，遍历key，调用自身set多个属性，设置链式操作
	if ( jQuery.type( key ) === "object" ) {
		chainable = true;
		for ( i in key ) {
			access( elems, fn, i, key[ i ], true, emptyGet, raw );
		}

	// value有值，则实现set功能，并设置链式操作
	} else if ( value !== undefined ) {
		chainable = true;
        //value不是函数，表示参数value就是要set的原始值
		if ( !jQuery.isFunction( value ) ) {
			raw = true;
		}

		if ( bulk ) {

			// key==null，并且value为原始值而不是函数，则执行fn，针对elems进行批量操作
			if ( raw ) {
				fn.call( elems, value );
				fn = null;

			// key == null，但value是函数，则先将批量操作函数化整为零，等待value函数执行的结果，分别针对每个elem，value依次执行
			} else {
				bulk = fn;
				fn = function( elem, key, value ) {
					return bulk.call( jQuery( elem ), value );
				};
			}
		}
        //为每个elem分别执行value函数，得到对应的value，对每个elem 执行fn
		if ( fn ) {
			for ( ; i < len; i++ ) {
				fn(
					elems[ i ], key, raw ?
					value :
					value.call( elems[ i ], i, fn( elems[ i ], key ) )
				);
			}
		}
	}
    //设置了chainable，则返回elem，支持链式调用
	if ( chainable ) {
		return elems;
	}

	// key==null，参数value==undefined，对elems执行fn，批量get数据
	if ( bulk ) {
		return fn.call( elems );
	}
    //否则，key是string，value为undefined，如果elems集合length>0,则对第一个元素，执行fn，get其属性key对应的值，否则，返回emptyGet
	return len ? fn( elems[ 0 ], key ) : emptyGet;
};
```
 ## 总结
 1. jquery.data将用户数据与内部数据分开，使用expando作为unique的key，将用户数据绑定到dom节点上，避免用户和其他模块误操作。支持get，set多种设置方式，并提供链式调用。兼容html5 data-*属性。并将其缓存在该节点的用户数据中，方便以后操作，避免多次操作dom，减少性能损耗。删除dom节点数据时，出于性能考虑，解除用户数据与dom节点之间的绑定，等待js回收机制释放内存。
 
2. jquery 1.x版本对于data的实现，是将data放在一个公共的缓存池jquery.cache中。而2.x/3.x版本使用Data类实现，通过dataUser和dataPriv两个对象操作dom节点的缓存。1.x版本中的jquery.cache是对外公开的，有被误写的可能，比如为jquery扩展一个cache对象，那么之前存储的数据都失去引用了。而2.x/3.x版本中只能通过dataUSer和dataPriv来操作cache，只对外提供了少数方法，数据都是私有的。

3. 1.x提供了一个jquery._data方法，仅供jquery内部来操作内部数据，但这只是一个约定式的约束。而2.x/3.x版本中使用dataPriv来操作内部数据，dataPriv完全是一个私有对象，对于数据的操作都封装在对象内部。jquery内部已经完全使用dataPriv代替jquery._data，只是由于兼容性，保留了jquery._data接口，官方注释称该方法会被deprecated。
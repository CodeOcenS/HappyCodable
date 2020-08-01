# HappyCodable
通过使用SourceKittenFramework去自动生成Codable代码, 使其使用起来让人更愉悦的Codable框架

## Codable的问题 ?

1. 不支持自定义某个属性的coding key, 一旦你有这种需求, 要么把所有的coding key手动实现一遍然后修改想要的coding key, 要么就得在decode的时候去修改JSONDecoder的设置, 及其不方便
2. 不支持忽略掉某些不能Codable的属性, 有这样的需求还是需要手动实现coding key才行
3. 不支持自动合成非RawRepresentable的Enum, 即使该Enum中所有值的子类型都是Codable也不行
4. decode的时候不支持多个coding key映射同一个属性
5. 难以调试, 虽然Codable是throw-catch的, 但是由于代码都是由编译器生成, 数据有问题的时候无法更近一步定位问题
6. 不能使用模型的默认值, 当decode 的数据缺失时无法使用定义里的默认值, 例如版本更新后, 服务端删掉了模型的某个过期字段, 这时候使用Codable只会throw数据缺失, 然后旧版本客户端都会陷入错误, 即使不用这个字段就版本客户端依旧是能正常工作的(只是显示的数据缺失而已), 这很明显是不合理的.
7. 不支持简单的类型映射, 比如转换0/1到false/true, "123"到Int的123或者反过来

### 而这些, 你全都可以用HappyCodable解决

## 用法

1. build the HappyCodable Command Line executable file
2. bring  executable file and HappyCodable Common Source Code to your project
3. add a run script in `build phases` before `compile sources` like:

```
${SRCROOT}/HappyCodableCommandLine ${SRCROOT}/Classes ${SRCROOT}/HappyCodable.generated.swift
```

4. 给struct/class添加HappyCodable, 然后编译一下:

```
struct Person: HappyCodable {
	var name: String = "abc"
	
	@Happy.codingKeys("🆔")
	var id: String = "abc"
	
	@Happy.codingKeys("secret_number", "age") // the first key will be the coding key
	var age: Int = 18
	
	@Happy.uncoding
	var secret_number: String = "3.1415" // Build fail if coded, in this case, we can "uncoding" it.
}
```

HappyCodableCommandLine会自动生成以下代码:

```
extension Person {
	enum CodingKeys: String, CodingKey {
		case name
		case id = "🆔"
		case age = "secret_number"
	}
	mutating
	func decode(happyFrom decoder: Decoder) throws {
		let container = try decoder.container(keyedBy: StringCodingKey.self)
		var errors = [Error]()
		
		do { self.name = try container.decode(default: self.name, key: "name", alterKeys: []) } catch { errors.append(error) }
		do { self.id = try container.decode(default: self.id, key: "🆔", alterKeys: []) } catch { errors.append(error) }
		do { self.age = try container.decode(default: self.age, key: "secret_number", alterKeys: ["age"]) } catch { errors.append(error) }
		
		if !Self.allowHappyDecodableSkipMissing, !errors.isEmpty {
			throw errors
		}
	}
	func encode(happyTo encoder: Encoder) throws {
		var container = encoder.container(keyedBy: CodingKeys.self)
		var errors = [Error]()
		do { try container.encodeIfPresent(self.name, forKey: .name) } catch { errors.append(error) }
		do { try container.encodeIfPresent(self.id, forKey: .id) } catch { errors.append(error) }
		do { try container.encodeIfPresent(self.age, forKey: .age) } catch { errors.append(error) }
		if !Self.allowHappyEncodableSafely, !errors.isEmpty {
			throw errors
		}
	}
}
```

还有非RawRepresentable Enum(你需要确保闭包里的类型都是Codable的):

```
enum EnumTest: HappyCodableEnum {
	case value(num: Int, name: String)
//	case call(() -> Void) // Build fails if added, since (() -> Void) can't be codable
	case name0(String)
	case name1(String, last: String)
	case name2(first: String, String)
	case name3(_ first: String, _ last: String)
}
```

生成的代码: 

```
extension EnumTest {
	init(from decoder: Decoder) throws {
		let container = try decoder.singleValueContainer()
		let content = try container.decode([String: [String: String]].self)
		let error = DecodingError.typeMismatch(EnumTest.self, DecodingError.Context(codingPath: [], debugDescription: ""))
		guard let name = content.keys.first else {
			throw error
		}
		let decoder = JSONDecoder()
		switch name {
			case ".value(num:name:)":
				guard
					let _0 = content[name]?["num"]?.data(using: .utf8),
					let _1 = content[name]?["name"]?.data(using: .utf8)
				else {
					throw error
				}
				
				self = .value(
					num: try decoder.decode((Int).self, from: _0),
					name: try decoder.decode((String).self, from: _1)
				)
			case ".name0(_:)":
				guard
					let _0 = content[name]?["$0"]?.data(using: .utf8)
				else {
					throw error
				}
				
				self = .name0(
					try decoder.decode((String).self, from: _0)
				)
			case ".name1(_:last:)":
				guard
					let _0 = content[name]?["$0"]?.data(using: .utf8),
					let _1 = content[name]?["last"]?.data(using: .utf8)
				else {
					throw error
				}
				
				self = .name1(
					try decoder.decode((String).self, from: _0),
					last: try decoder.decode((String).self, from: _1)
				)
			case ".name2(first:_:)":
				guard
					let _0 = content[name]?["first"]?.data(using: .utf8),
					let _1 = content[name]?["$1"]?.data(using: .utf8)
				else {
					throw error
				}
				
				self = .name2(
					first: try decoder.decode((String).self, from: _0),
					try decoder.decode((String).self, from: _1)
				)
			case ".name3(_:_:)":
				guard
					let _0 = content[name]?["first"]?.data(using: .utf8),
					let _1 = content[name]?["last"]?.data(using: .utf8)
				else {
					throw error
				}
				
				self = .name3(
					try decoder.decode((String).self, from: _0),
					try decoder.decode((String).self, from: _1)
				)
		default:
			throw error
		}
	}
	func encode(to encoder: Encoder) throws {
		var container = encoder.singleValueContainer()
		let encoder = JSONEncoder()
		switch self {
			case let .value(_0, _1):
				try container.encode([
					".value(num:name:)": [
						"num": String(data: try encoder.encode(_0), encoding: .utf8),
						"name": String(data: try encoder.encode(_1), encoding: .utf8)
					]
				])
			case let .name0(_0):
				try container.encode([
					".name0(_:)": [
						"$0": String(data: try encoder.encode(_0), encoding: .utf8)
					]
				])
			case let .name1(_0, _1):
				try container.encode([
					".name1(_:last:)": [
						"$0": String(data: try encoder.encode(_0), encoding: .utf8),
						"last": String(data: try encoder.encode(_1), encoding: .utf8)
					]
				])
			case let .name2(_0, _1):
				try container.encode([
					".name2(first:_:)": [
						"first": String(data: try encoder.encode(_0), encoding: .utf8),
						"$1": String(data: try encoder.encode(_1), encoding: .utf8)
					]
				])
			case let .name3(_0, _1):
				try container.encode([
					".name3(_:_:)": [
						"first": String(data: try encoder.encode(_0), encoding: .utf8),
						"last": String(data: try encoder.encode(_1), encoding: .utf8)
					]
				])
		}
	}
}
```

## 局限性

1. 因为HappyCodable是通过生成一个文件给所有需要的类型生成Codable方法的extension, 因此没法用于private的模型, 同理也没法用于定义在方法里的模型:

   ```
   func getNumber() {
   	struct Package: Codable {
   		let result: Int
   	}
   }
   ```

2. HappyCodable要求实现一个 `init()` 方法创建一个默认的变量(HappyCodableEnum不需要), 然后再通过Codable给需要的属性编码, 所以它要求编码的属性都是mutable的, 像上面的Package这种只读模型就没法用了.


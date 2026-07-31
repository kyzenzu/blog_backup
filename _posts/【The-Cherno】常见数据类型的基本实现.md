---
title: 【The Cherno】常见数据类型的基本实现
date: 2026-01-31 14:56:00
tags:
  - C++
---

对于常见的数据类型，`STL`为了保证健壮性会对定义做非常多保护，导致难以快速的找到数据类型的核心。所以在此对常见的数据类型进行简单的实现，保留核心功能。

### Array

~~~cpp
template<typename T, int N>
class Array {
private:
	T m_Array[N];
public:
	int Size() const { return N; }
};
~~~

~~~cpp
int main() {
	Array<std::string, 5> a;
    a.Size();
}
~~~

### String

~~~cpp
class String {
	friend std::ostream& operator<<(std::ostream&, const String&);
private:
	char* m_Buffer;
	unsigned int m_Size;
public:
	String(const char* str) {
		m_Size = strlen(str);
		m_Buffer = new char[m_Size + 1];
		strcpy(m_Buffer, str);
	}
	String(const String& other) : m_Size(other.m_Size) {
		m_Buffer = new char[m_Size + 1];
		strcpy(m_Buffer, other.m_Buffer);
	}
	~String() { delete m_Buffer; }
	char& operator[](unsigned int index) { return m_Buffer[index]; }
};
std::ostream& operator<<(std::ostream& os, const String& str) {
	os << str.m_Buffer;
	return os;
}
~~~

~~~cpp
int main() {
	String str = "Cherno";
	String str1 = str;
	str1[2] = 'a';
	std::cout << str << std::endl;
	std::cout << str1 << std::endl;
}
~~~

### ScopedPtr

~~~cpp
class ScopedPtr {
private:
	Entity* m_Ptr;
public:
	ScopedPtr(Entity* ptr) : m_Ptr(ptr) {}
	~ScopedPtr() { delete m_Ptr; }
	Entity* operator->() { return m_Ptr; }
	Entity* operator->() const { return m_Ptr; }
};
~~~

~~~cpp
struct Entity {
	int x, y;
	void Print() {
		std::cout << x << " " << y << std::endl;
	}
};
int main() {
	const ScopedPtr e = new Entity;
	e->Print();
}
~~~


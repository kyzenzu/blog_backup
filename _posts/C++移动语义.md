---
title: C++移动语义
date: 2023-4-1
tags:
  - C/C++
  - C++
---

# 移动语义

~~~C++
#include <iostream>

class String
{
public:
	String() = default;
	String(const char* string) {
		printf("Create...\n");
		m_Size = (uint32_t)strlen(string);
		m_Str = new char[m_Size];
		memcpy(m_Str, string, m_Size);
	}
	String(const String& other) {
		printf("Copy...\n");
		m_Size = other.m_Size;
		m_Str = new char[m_Size];
		memcpy(m_Str, other.m_Str, m_Size);
	}
	String(String&& other) noexcept {
		printf("Move...\n");
		m_Size = other.m_Size;
		m_Str = other.m_Str;
		other.m_Size = 0;
		other.m_Str = nullptr;
	}
	String& operator=(String&& other) noexcept {
		printf("Move...\n");
		if (this != &other) {
			delete[] this->m_Str;

			m_Size = other.m_Size;
			m_Str = other.m_Str;
			other.m_Size = 0;
			other.m_Str = nullptr;
		}
		return *this;
	}
	~String() {
		printf("Destroyed...\n");
		delete m_Str;
	}
	void Print() {
		for (uint32_t i = 0; i < m_Size; i++) {
			printf("%c", m_Str[i]);
		}
		printf("\n");
	}
private:
	char* m_Str;
	uint32_t m_Size;
};

class Entity
{
public:
	Entity(const String& name) : m_Name(name) {

	}
	Entity(String&& name) : m_Name(std::move(name)) {

	}
	void PrintName() {
		m_Name.Print();
	}
private:
	String m_Name;
};

int main()
{
	//Entity entity("Cherno");
	//entity.PrintName();
	//String string = "Hello";
	//String des(std::move(string));

	String apple = "Apple";
	String dest;
	apple.Print();
	dest.Print();
	printf("----\n");
	dest = std::move(apple);
	apple.Print();
	dest.Print();
	std::cin.get();
}
~~~


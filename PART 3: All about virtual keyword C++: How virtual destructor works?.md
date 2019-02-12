### Recap
- We have already learned in our earlier post that how [virtual function]() & [virtual class]() works.
- We left topic virtual destructor in these articles which we will learn now.
- As usual, before learning about virtual destructor we will see why it needed at first place.
### Why we need a virtual destructor?
- We will understand this with our earlier example(slightly twisted) which was as follow:
```
class protocol_t{
	private:
		uint8_t *_type;
		// storage ...
	public:
    		protocol_t() { _type = new uint8_t; }
    		~protocol_t() { cout<<"~protocol_t"; delete _type; }

		virtual void authenticate(){};
		virtual void connect(){};
		// operations ...
};

class wifi_t : public protocol_t{
	private:
		char *_pass;
		// storage ...
	public:
    		wifi_t() { _pass = new char[15]; }
    		~wifi_t() { cout<<"~wifi_t"; delete _pass; }

		virtual void authenticate(){};
		virtual void connect(){};
		// operations ...
};

class bluetooth_t : public protocol_t{
	private:
		char *_pass;
		// storage ...
	public:
    		bluetooth_t() { _pass = new char[15]; }
    		~bluetooth_t(){ cout<<"~wifi_t"; delete _pass; }

		virtual void authenticate(){};
		virtual void connect(){};
		// operations ...
};

void makeConnection(protocol_t *protocol)
{
	protocol->authenticate();
	protocol->connect();
  
  	// Do some tx & rx
  
  	delete protocol;
}	


int main()
{
	int prot_type = rand()%2;
  
	makeConnection( (prot_type) ? static_cast<protocol_t*>(new wifi_t) : static_cast<protocol_t*>(new bluetooth_t) );	

	return 0;
}
```
- So, we have created `wifi_t` & `bluetooth_t` objects dynamically in `main()` and passed it to function `makeConnection()`.
- An objects of `wifi_t`, `bluetooth_t` & `protocol_t` also uses heap memory at construction & destruction time.
- Well, this code compiles & runs fine without any error. But when you run above code, at the time of `delete protocol` line it alwasy call destructor of `protocol_t` which you can verify by `~protocol_t` print on console.
- We are freeing only sub-object resources which is `protocol_t` in call of `~protocol_t()` destructor. This means that there is memory leak as we are not freeing heap memory resource of object pointed by pointer `protocol_t` in function `makeConnection()`.
- We even dont know the type of object `protocol_t` pointer pointed to.
- Virtual destructors are there to solve this problem. What we have to do is that
```
virtual ~protocol_t() { cout<<"~protocol_t"; delete _type; }
```
- Put keyword virtual in front of `~protocol_t()` destructor. It will call the destructor of object pointed by pointer `protocol_t` & then call `~protocol_t()` destructor.

### How virtual destructor works?

- The question is that how our destructor of derived class called. Answer is simple it just overridden in virtual table. Lets understand it with assumption that our pointer `protocol` points to object of type `wifi_t`.
```
protocol_t *protocol = new wifi_t;
delete protocol;
```
- Here is memory layout of object `wifi_t`
```
|                                |          
|--------------------------------| <------ wifi_t object memory layout
|  protocol_t::_type             |          
|--------------------------------|          
|  protocol_t::_vptr_protocol_t  |----------|
|--------------------------------|          |----------|-------------------------|
|  wifi_t::_pass                 |                     |   type_info wifi_t      |
|--------------------------------|                     |-------------------------|
|                                |                     |   wifi_t::authenticate  |
|                                |                     |-------------------------|
|                                |                     |   wifi_t::connect       |
|                                |                     |-------------------------|
|                                |                     |   wifi_t::~wifi_t       |
|                                |                     |-------------------------|
```
- Now, the statement `delete protocol;` will be transformed into
```
( * protocol->vptr[ 3 ])( protocol ); 
```
- Till here it was simple for us to understand how things are working because this is virtual function mechanism. But real magic comes when destructor of `protocol_t` will be called.
- This is again augmented code by compiler in derived class destructor & probably would become:
```
~wifi_t() { 
	cout<<"~wifi_t"; 
	delete this->_pass; // Compiler will refer variable from start of object, so implicit 'this'
	
	// Compiler augmented code ----------------------------------------------------
	// Rewire virtual table
	this->vptr[0] = &type_info_protocol_t;
	this->vptr[1] = &protocol_t::authenticate;
	this->vptr[2] = &protocol_t::connect;
	this->vptr[3] = &protocol_t::~protocol_t();
	
	// Call to base class destructor
	protocol_t::~protocol_t(this); 
}
```
- The process of destructing an object takes more operations than those you write inside the body of the destructor. When the compiler generates the code for the destructor, it adds extra code both before and after the user defined code. Here we have only taken after code for sake of understanding.
- Same process will happen no mattern how long tree up there is.

### Tricky example
```
struct base {
   virtual ~base() { f(); }
   virtual void f() { std::cout << "base"; }
};
struct derived : base {
   void f() { std::cout << "derived"; }
};
int main() {
   base * p = new derived;
   delete p;
}
```
- Output is `base`. Because standard mandates that the runtime type of the object is that of the class being constructed/destructed at this time, even if the original object that is being constructed/destructed is of a derived type.

### Reference 
- http://www.avabodh.com/cxxin/virtualbase.html
- https://stackoverflow.com/questions/7750280/how-does-virtual-destructor-work-in-c
- Book: Inside C++ Object Model By Lippman
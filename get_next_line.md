# get_next_line
A function that reads one line from a specified file descriptor every time it is called. 
In this project we learn about memory leaks, static variables, the read function and the -D preprocessor flag.

### Index:
+ [Read and open function](#read-and-open-function)
+ [What is a memory leak?](#what-is-a-memory-leak)
+ [Debugging tools for memory leaks](#debugging-tools-for-memory-leaks) 
+ [Why we use BUFFERSIZE](#why-we-use-buffersize) 
+ [Static variable](#static-variable)
+ [Preprocessor -D flag](#preprocessor--d-flag)
+ [Tips on writing get_next_line](#tips-on-writing-get_next_line)

## Read and open function
[Read()](https://linux.die.net/man/2/read) reads from the given file descriptor in the first argument, into a buffer given by the second and how much is read is determined by the third argument. The return will be how many characters has been read or -1 on error. `read(fd, buffer, count)`. The read function continues where it left of every time you call it.

[Open()](https://man7.org/linux/man-pages/man2/open.2.html) opens a file to the given file descriptor. And then this file descriptor can be used as an argument to get_next_line. You can use flags with this for additional behaviour, like O_CREAT, which will create the file with the given name if it doesn't exist already. 
```c
int32_t fd

fd = open("testfile", 0)
```


## What is a memory leak?
A memory leak happens when you lose the reference to dynamically allocated memory. Like let's say you malloc something and then you replace 
the address of what that pointer is pointing to, you wouldn't be able to control that memory anymore and it will still be taking up space in memory. 
```c
int32_t main(void)
{
  char	*ptr;
  char	a;
  
  ptr = malloc(1); // allocates memory on the heap and stores that address in ptr. 
  ptr = &a; // Now you store the address of a in ptr and the reference to the previous allocation is lost in the void!
}
```
Memory leaks reduce the amount of memory left for other applications to utilise and can reduce performance or even cause your system to crash. 
Memory allocation and freeing is done for you when you initialise variables on the stack. It keeps track of the order of execution of the functions. Whenever you exit a function and return to the previous function, the memory will automatically be freed. This is not the case when you allocate something with any function utilising malloc() and you are allocating memory on the heap. This does give you extra flexibility, as your allocation won't be lost when you return from the function where it was allocated.

### Debugging tools for memory leaks
+ [Valgrind](https://www.youtube.com/watch?v=bb1bTJtgXrI) - `valgrind --leaks-check=full ./executable` 
+ [AdressSanitizer (ASan)](https://www.osc.edu/resources/getting_started/howto/howto_use_address_sanitizer) - `gcc -Wall -Werror -Wextra test.c -fsanitize=address -o test` 
+ Leaks - `leaks --atExit --./executable`

## Why we use BUFFERSIZE
### What it means 
You could just read everything character by character until you reach a newline character and return that line, that would be the easy solution. However, the tricky part to this project is that we have to read everything with a dynamic BUFFERSIZE. Meaning that the amount that is read at a time by the read function, should be changeable. For example, a sentence could be 10 characters long and if the BUFFERSIZE is set to 4, that means it will take three itterations to read that particular sentence, if it is set to 10, it would take only one itteration.
### Why we use it
We use it so for different contexts we can consider how a certain buffersize will impact the memory usage vs speed of your program. 
The bigger the BUFFERSIZE, the faster your program will execute, but the more memory space it will occupy. 
So for instance, you want to read an entire book. In this case, it would be really slow to go through this character by character. And you would probably want to set the BUFFERSIZE quite differently than in the case where you read just one paragraph.

```c
char *cheat_get_next_line(int32_t fd) // This is the cheat version that reads character by character
{
	char	*line; // Where we store the line to be returned
	int32_t	buflen;
	ssize_t	i;

	i = -1;
	line = malloc(10000);
	while (line[i++] != '\n') // We want to read until we reach a newline character
	{
		buflen = read(fd, &line[i], 1); // the return of line is -1 on error
		if (i > 0 && buflen == 0) // checking for the end of the file
			return (line);
		if (buflen <= 0) // checking for errors
			return (free(line), NULL);
	}
	line[i] = '\0'; // always make sure to null terminate your strings!
	return (line);
}
```
## Static variable
A [static variable](https://www.geeksforgeeks.org/static-variables-in-c/) is essentially like a global variable, but only within the scope of one function. Meaning that if you return to the function the static variable is delcared, it will have retained its previous value. A static variable is stored in the [BSS data segment](https://www.geeksforgeeks.org/memory-layout-of-c-program/), where global variables are also stored. 

## Preprocessor -D flag
[Preprocessor flags](https://gcc.gnu.org/onlinedocs/gcc/Preprocessor-Options.html) control the preprocessor, which is ran before compilation of source files. The -D flag defines a macro. In the same way a #define would work in a header. 
` gcc -Wall -Werror -Wextra getnextline.c -DBUFFERSIZE=42 -o getnextline` 
This does the same as: 
```
#ifndef GET_NEXT_LINE_H
# define GET_NEXT_LINE_H

# ifndef BUFFER_SIZE
#  define BUFFER_SIZE 42
# endif

#endif 
```

## Tips on writing get_next_line
I think the biggest problem you encounter with writing get_next_line is when you read a bit after the newline character or when encountering multiple newline characters. Let's say your BUFFERSIZE=6 and you are reading this text file: 
```
Something\n
\n
Is here\n
```
The first itteration you will have read `Someth` and the next time you will read `ing\n\nI`. So when you join these values, you will end up with `Something\n\nI`. Now you will want to seperate `Something\n` from everything else, as that is the value you want to return, but you don't want to lose the information afther the newline character! The next time you call your function, the read function will continue where it has left off, so that information will be lost if not saved somewhere. This is where the static variable comes in! You can use this to retain this information for the next time you call the function. But since there is already another newline that has been read, you have to be extra careful you don't read any more in this case and the line is further seperated. 

_uhhhhhh make it what? before you start coding, make a sketch... Sketch out howww (pfffff... idk?!) How the static variables behave when the function is called multiple times. Be aware of error cases such as invalid file and how to handle them (not handle but detect them). uhhhh... aannd take good care that you return especially the last line. Learn to manage your buffer, because unlike in the cheat-version you actually have to do that xdd.
Sidenote: maybe read the subject too ~~(and not compile into an archive like before ;) )~~_ - [Clemens](https://profile.intra.42.fr/users/cdahlhof) 


_"Do not malloc a static variable or Russian maffia will come after you"_ - [Kuba](https://github.com/jakubkaczmarski)

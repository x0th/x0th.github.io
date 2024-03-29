---
layout: post
title: Analyzing curl with IKOS, KLEE and CBMC
categories: Program_analysis
---

This is a report for a project that I did for CSC 255 Software Analysis & Improvement while in University of Rochester. Thanks to Professors Sreepathi Pai's permission and encourangement, I am able to post it. The report was edited minorly to translate it to the Markdown language.
The project involved finding and analyzing bug(s) in the curl library using analysis tools such as IKOS, KLEE and CBMC. The code for the project is available [here](https://github.com/x0th/analyzing-curl).

---
## 1. **The bug**

The bug I chose to look at was reported as [CVE-2018-16890](https://curl.se/docs/CVE-2018-16890.html) and was fixed in curl version 7.64.0. The version of curl that I am looking into is 7.63.0, with the bug still present.

The bug itself is a heap buffer out-of-bounds read during memcpy in the function ntlm_decode_type2_target() (see the function below). The bug is simple: the function fills struct ntlmdata ntlm based on the contents of an unsigned char buffer of size size. The function gets two values from the buffer, the length of data to be copied from the buffer (target_info_len) and the offset of data to be copied within the buffer (target_info_offset). Then, it copies target_info_len bytes of the buffer starting at target_info_offset using memcpy. While the function has an out-of-bounds check, it is insufficient.

```c
static CURLcode ntlm_decode_type2_target(struct Curl_easy *data,
										 unsigned char *buffer,
										 size_t size,
										 struct ntlmdata *ntlm)
{
	unsigned short target_info_len = 0;
	unsigned int target_info_offset = 0;
#if defined(CURL_DISABLE_VERBOSE_STRINGS)
	(void) data;
#endif

	if(size >= 48) {
		target_info_len = Curl_read16_le(&buffer[40]);
		target_info_offset = Curl_read32_le(&buffer[44]);
		if(target_info_len > 0) {
			if(((target_info_offset + target_info_len) > size) ||
				(target_info_offset < 48)) {
					return CURLE_BAD_CONTENT_ENCODING;
			}

			ntlm->target_info = malloc(target_info_len);
			if(!ntlm->target_info)
				return CURLE_OUT_OF_MEMORY;

			assert(target_info_offset >= size);
			memcpy(ntlm->target_info, &buffer[target_info_offset], target_info_len);
		}
	}

	ntlm->target_info_len = target_info_len;

	return CURLE_OK;
}
```

## 2. **The analysis method**

I used IKOS, KLEE and CBMC as the tools for analysis. I combine them into a single python script, runtools.py, which runs the three tools on a simplified version of ntlm.c (lib/vauth/ntlm.c in the library source). See section 3 on why I chose not to run the tools on the original source file.

First, the script runs KLEE, with symbolic versions of buff and ntlm. For some reason, KLEE will not report the memcpy error (though it will report other errors) if you do not make ntlm symbolic. While in the original program ntlm was meant to be filled by the function, not before, I would argue that it does not matter since the bug is fully dependent on the contents of buff so making ntlm symbolic in order to get better analysis is a fair assumption. If the error you were looking for (memcpy in this instance) is found, the script saves the test case generated by KLEE for this error.

Second, the script runs IKOS with the test case generated by KLEE. While it is possible to run IKOS on a source file with uninitialized buff, IKOS will then miss the memcpy bug. So, here IKOS is used as an assurance tool that the test case, indeed, produces the desired error.

Lastly, the script runs CBMC with buff filled with nondeterministic unsigned chars. Here, CBMC is also used as an assurance that the error exists – it is run without --trace, so it will not produce a test case.
The output of the script on my machine is as follows:

![alt text](/images/analyzing_curl_1.png "Bug output")

I also run the script on a simplified fixed version of ntlm_decode_type2_target() (file ntlm_7_64.c) (you can see the fix in this [git commit](https://github.com/curl/curl/commit/b780b30d1377adb10bbe774835f49e9b237fb9bb)):

![alt text](/images/analyzing_curl_2.png "Fixed output")

## 3. Problems and limitations

First off, I feel a need to justify working on a simplified source file. There are two main reasons for this.

First, because the bug is in libcurl, which is basically monolithic, after the source file is compiled, the remainder of the files have to be dynamically linked in for the tools to correctly work. KLEE, in particular, really could not see all the linked in bytecode files (I used the --link-llvm-lib option). Even IKOS, which has a wonderful ikos-scan script that can run ./configure and make to compile the whole library to be IKOS-compatible struggled when it came to analysis of specific library functions instead of the fully-linked curl.

Second, because I chose to verify that a bug exists instead of searching for one, I think making a simplified version of the function is perfectly reasonable. One can easily imagine how the tools would be used as a part of a test suite, even with the given simplifications. However, that would probably require tinkering with the Makefile and I am not familiar enough with the codebase to do that. When I tried to make the tools work by hand, even on source files that were linked fully, they lost precision (in particular, IKOS and KLEE). And trying to hack together a test in which the bug becomes visible to the tools does not make sense to me. If I was working on an actual test suite that is supposed to find bugs this is exactly what I would do – let the tools make all the test cases instead of thinking of a test case that would break the program – after all, that is the whole point of using non-concrete analysis options.

I would also like to talk about other limitations of the tools. Another bug I tried to analyze was [CVE-2016-9586](https://curl.se/docs/CVE-2016-9586.html). Curl have their own printf implementation, which could lead to buffer overflow if a large enough (in terms of characters) float number was entered. While I was able to make it work, neither KLEE nor IKOS could see the bug unless specific float values were entered. CBMC was also not able to see the bug because of loop unrolling. Specifically, part of the printf implementation is the following:
```c
static long dprintf_DollarString(char *input, char **end)
{
	int number=0;
	while(ISDIGIT(*input)) {
		number *= 10;
		number += *input-'0';
		input++;
	}
	if(number && ('$'==*input++)) {
		*end = input;
		return number;
	}
	return 0;
}
```
Here, no matter how I tried, CBMC was not able to unroll the while loop. And, not unrolling the loop at all lead to no bug (since the buffer was not getting filled).

## 4. Conclusions

Overall, I would argue that the goal of the project was met. I show how the work several tools, IKOS, KLEE and CBMC can be automated with a simple semi-general script to verify existing bugs. I also believe that this suite can be used to find new bugs, although I do think that it would take a considerable amount of work to make it operable on a large library. I also show the limitations of the tools and how some examples can produce either too unprecise (effectively useless) results or no results at all.
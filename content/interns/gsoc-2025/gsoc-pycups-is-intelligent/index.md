+++
title = 'GSOC: PyCups3 is intelligent?'
date = 2025-08-18T11:00:00+05:30
images = ['openprinting.png']
keywords = ['OpenPrinting', 'Python', 'PyCups', 'GSOC', '2025']
tags = ['OpenPrinting', 'GSOC', '2025']
+++

> Is that really so? PyCups3 is intelligent?

Well, short answer is YES. It is intelligent.

> Damn, sweetheart, you made an AI for PyCups?

Not everything that's intelligent is an AI.

An imaginary conversation, between me and my girlfriend. Now, what's the twist here is, PyCups3 is actually very very intelligent. So, from my last blog post, I detailed how, PyCups2 got lost due to the lack of upgrades and implmentation of new APIs.

This is where my implementation of PyCups using CFFI shines. Let's say, there is an API in `libcups3`, that is not yet implemented _Pythonically_ in `PyCups3`. Does that mean you're blocked? You cannot use that function anymore? Not with `PyCups3`. Let's say there is an API in `libcups3`, which is used to get the password that's given by the user for authentication, and it's not yet implemented in `PyCups` _Pythonically_. How can you use it? You'll simply do 3 steps,

```python
>> import cups # step 1 import all the necessary modules
>> from cups import lib, ffi  # import the ffi bridges of backend
>> from cups.utils import _bytes_to_value  # import a helper to convert a C char into a Python string
>> conn = cups.Connection()  # step 2: create an http connection
>> _bytes_to_value(lib.cupsGetPassword(b"Enter Password", conn.http, b"GET", ffi.NULL))  # step 3: call the API
```

This will give you an output like this.

```bash
Enter Password ********
'test1234'
```

Do you think this is very simple and easy? Well, let's take this one level up. Let's do some custom ipp requests. (Inspired from PyCups2)

Let's make raw IPP request to get all the attributes associated with a printer destination.

```python
>> import cups
>> from cups.types import IPPRequest
>> from cups.enums import IPPOp, IPPTag
>> from cups import ffi, lib
>> conn = cups.Connection()
>> req = IPPRequest.cffi_new(IPPOp.GET_PRINTER_ATTRIBUTES)
>> req.addString(group=IPPTag.OPERATION, value_tag=IPPTag.URI, name="printer-uri",
    value="ipp://localhost/printers/<your-printer-name>")
>> res = IPPRequest(lib.cupsDoRequest(conn.http, req.ffi_value, b"/"))
>> if res is not None:
     for attr in res.attributes:
       print(f"{attr.name}: {attr.values}")
```

Skipping the imports, this is just 5 lines of code. Comparing this [with the implementation in `PyCups2`](https://github.com/OpenPrinting/pycups/blob/master/cupsconnection.c#L3208-L3405).

```c
static PyObject *
Connection_getPrinterAttributes (Connection *self, PyObject *args,
				 PyObject *kwds)
{
  PyObject *ret;
  PyObject *nameobj = NULL;
  char *name;
  PyObject *uriobj = NULL;
  char *uri;
  PyObject *requested_attrs = NULL;
  char **attrs = NULL; /* initialised to calm compiler */
  size_t n_attrs = 0; /* initialised to calm compiler */
  ipp_t *request, *answer;
  ipp_attribute_t *attr;
  char consuri[HTTP_MAX_URI];
  int i;
  static char *kwlist[] = { "name", "uri", "requested_attributes", NULL };

  if (!PyArg_ParseTupleAndKeywords (args, kwds, "|OOO", kwlist,
				    &nameobj, &uriobj, &requested_attrs))
    return NULL;

  if (nameobj && uriobj) {
    PyErr_SetString (PyExc_RuntimeError,
		     "name or uri must be specified but not both");
    return NULL;
  }

  if (nameobj) {
    if (UTF8_from_PyObj (&name, nameobj) == NULL)
      return NULL;
  } else if (uriobj) {
    if (UTF8_from_PyObj (&uri, uriobj) == NULL)
      return NULL;
  } else {
    PyErr_SetString (PyExc_RuntimeError,
		     "name or uri must be specified");
    return NULL;
  }

  if (requested_attrs) {
    if (get_requested_attrs (requested_attrs, &n_attrs, &attrs) == -1) {
      if (nameobj)
	free (name);
      else if (uriobj)
	free (uri);
      return NULL;
    }
  }

  debugprintf ("-> Connection_getPrinterAttributes(%s)\n",
	       nameobj ? name : uri);

  if (nameobj) {
    construct_uri (consuri, sizeof (consuri),
		   "ipp://localhost/printers/", name);
    uri = consuri;
  }

  for (i = 0; i < 2; i++) {
    request = ippNewRequest (IPP_GET_PRINTER_ATTRIBUTES);
    ippAddString (request, IPP_TAG_OPERATION, IPP_TAG_URI,
		  "printer-uri", NULL, uri);
    if (requested_attrs)
      ippAddStrings (request, IPP_TAG_OPERATION, IPP_TAG_KEYWORD,
		     "requested-attributes", n_attrs, NULL,
		     (const char **) attrs);
    debugprintf ("trying request with uri %s\n", uri);
    Connection_begin_allow_threads (self);
    answer = cupsDoRequest (self->http, request, "/");
    Connection_end_allow_threads (self);
    if (answer && ippGetStatusCode (answer) == IPP_NOT_POSSIBLE) {
      ippDelete (answer);
      if (uriobj)
	break;

      // Perhaps it's a class, not a printer.
      construct_uri (consuri, sizeof (consuri),
		     "ipp://localhost/classes/", name);
    } else break;
  }

  if (nameobj)
    free (name);

  if (uriobj)
    free (uri);

  if (requested_attrs)
    free_requested_attrs (n_attrs, attrs);

  if (!answer || ippGetStatusCode (answer) > IPP_OK_CONFLICT) {
    set_ipp_error (answer ? ippGetStatusCode (answer) : cupsLastError (),
		   answer ? NULL : cupsLastErrorString ());
    if (answer)
      ippDelete (answer);

    debugprintf ("<- Connection_getPrinterAttributes() (error)\n");
    return NULL;
  }

  ret = PyDict_New ();
  for (attr = ippFirstAttribute (answer); attr; attr = ippNextAttribute (answer)) {
    while (attr && ippGetGroupTag (attr) != IPP_TAG_PRINTER)
      attr = ippNextAttribute (answer);

    if (!attr)
      break;

    for (; attr && ippGetGroupTag (attr) == IPP_TAG_PRINTER;
	 attr = ippNextAttribute (answer)) {
      size_t namelen = strlen (ippGetName (attr));
      int is_list = ippGetCount (attr) > 1;

      debugprintf ("Attribute: %s\n", ippGetName (attr));
      // job-sheets-default is special, since it is always two values.
      // Make it a tuple.
      if (!strcmp (ippGetName (attr), "job-sheets-default") &&
	  ippGetValueTag (attr) == IPP_TAG_NAME) {
	PyObject *startobj, *endobj, *tuple;
	const char *start, *end;
	start = ippGetString (attr, 0, NULL);
	if (ippGetCount (attr) >= 2)
	  end = ippGetString (attr, 1, NULL);
	else
	  end = "";

	startobj = PyObj_from_UTF8 (start);
	endobj = PyObj_from_UTF8 (end);
	tuple = Py_BuildValue ("(OO)", startobj, endobj);
	Py_DECREF (startobj);
	Py_DECREF (endobj);
	PyDict_SetItemString (ret, "job-sheets-default", tuple);
	Py_DECREF (tuple);
	continue;
      }

      // Check for '-supported' suffix.  Any xxx-supported attribute
      // that is a text type must be a list.
      //
      // Also check for attributes that are known to allow multiple
      // string values, and make them lists.
      if (!is_list && namelen > 10) {
	const char *multivalue_options[] =
	  {
	    "notify-events-default",
	    "requesting-user-name-allowed",
	    "requesting-user-name-denied",
	    "printer-state-reasons",
	    "marker-colors",
	    "marker-names",
	    "marker-types",
	    "marker-levels",
	    "member-names",
	    NULL
	  };

	switch (ippGetValueTag (attr)) {
	case IPP_TAG_NAME:
	case IPP_TAG_TEXT:
	case IPP_TAG_KEYWORD:
	case IPP_TAG_URI:
	case IPP_TAG_CHARSET:
	case IPP_TAG_MIMETYPE:
	case IPP_TAG_LANGUAGE:
	case IPP_TAG_ENUM:
	case IPP_TAG_INTEGER:
	case IPP_TAG_RESOLUTION:
	  is_list = !strcmp (ippGetName (attr) + namelen - 10, "-supported");

	  if (!is_list) {
	    const char **opt;
	    for (opt = multivalue_options; !is_list && *opt; opt++)
	      is_list = !strcmp (ippGetName (attr), *opt);
	  }

	default:
	  break;
	}
      }

      if (is_list) {
	PyObject *list = PyList_from_attr_values (attr);
	PyDict_SetItemString (ret, ippGetName (attr), list);
	Py_DECREF (list);
      } else {
	PyObject *val = PyObject_from_attr_value (attr, i);
	PyDict_SetItemString (ret, ippGetName (attr), val);
      }
    }

    if (!attr)
      break;
  }

  debugprintf ("<- Connection_getPrinterAttributes() = dict\n");
  return ret;
}
```

Looks like 200 lines of code. But, how is that possible? It's due the foundation of `PyCups3`. From day 1, I tried to represent every single possible C structure in Python. And this is where the `@property` decorator is coming handy the most. I recently made a post about this in LinkedIn. If someone is interested on how I am using this decorator to handle live C data from python directly, they can check it out [here](https://www.linkedin.com/posts/soumyadghosh_why-i-fell-in-love-with-property-while-activity-7356164166472118275-ysfk?utm_source=share&utm_medium=member_desktop&rcm=ACoAAEb88BYBvz0xFt2Nk4WyZerJF0FdcaAdvog).

BTW, a small change from that last, I am probably going to say good bye to `dataclass`. This is an initial decision, waiting for a discussion with my mentor for the final blow. Once done, I'll keep you updated in another blog post about why I did so.


Coming to the comparison, a lot of work was needed to be done by `PyCups2` just to calculate the attributes, their values etc. But, I am doing those calculations here on demand, when needed by the user. The member `values` of the struct `ipp_attribute_t` is calling the different ipp getters ([`ippGetString`](https://openprinting.github.io/cups/libcups/cupspm.html#ippGetString), [`ippGetInteger`](https://openprinting.github.io/cups/libcups/cupspm.html#ippGetInteger) etc.) whenever the user is asking for the value.

While discussing about ippAttributes, I want to highlight one thing, about my mentor Callahan Covacs. This one example, prove again, seniority has its own glory.

If an ippAttribute is a Date, the getter to get that value is [`ippGetDate`](https://openprinting.github.io/cups/libcups/cupspm.html#ippGetDate). The value it returned is a 7 bytes hex date value. Now, I wanted to parse it as a datetime for a clear representation of the value. And here is the clever idea of Callahan. He showed me how we can use the struct library, to unpack the hex value in the form raw binary data to integers.

```python
struct.unpack(">H5B", date_hex_bytes)
```

Now, the data is in Big Endian format (most significant byte is placed first). The first 2 bytes represents the year, and next 5 bytes, month, date, hour, minutes and seconds. struct returns this as a tuple, and if we unpack the tuple inside the constructor of `datetime`, it'll create a proper `datetime` instance, which can be easily used in a python friendly way.

Thank you so much Callahan. I am very grateful to be your mentee for this GSoC 2025.

All the data from the attributes, comes when needed, no caching inside python, no data mutation either. `PyCups2` also did the similar thing, but only with the API callings, not with the underlying data.

PS: If you want to give all this a try, you find the source code [here](https://github.com/soumyaDghosh/pycups/tree/feat-refactor) in the feat-refactor branch (yet to be merged).

**Before doing this, make sure you've `libcups3` installed in your system.**

You can install it directly using uv
```bash
uv venv
uv pip install git+https://github.com/soumyaDghosh/pycups@feat-refactor
```

or using pip

```bash
source <venv-path>/bin/activate
pip install git+https://github.com/soumyaDghosh/pycups@feat-refactor
```

There are lot more technical jargons related to the project and the work is still going on. So, stay tuned... or as I said, stay caffeinated. ☕

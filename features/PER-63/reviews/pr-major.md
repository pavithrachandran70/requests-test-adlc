**Trigger Condition:** Same three-arg FileNotFoundError constructor issue as above (missing key path raise)

**Guard Snippet:** Use FileNotFoundError(message) and set .filename attribute explicitly, or use two-arg form FileNotFoundError(errno, message) which Python will map to .errno and .strerror, then set .filename separately

**Potential Consequence:** Runtime TypeError or malformed exception at call site; AC2's contract (.filename holds missing key path) breaks

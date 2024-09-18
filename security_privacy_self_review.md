### What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?
This feature does not expose any new information.

### Do features in your specification expose the minimum amount of information necessary to enable their intended uses?
This feature does not expose any new informnation.

### How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?
This feature does not deal with personal information.

### How do the features in your specification deal with sensitive information?
This feature does not deal with sensitive information>

### Do the features in your specification introduce new state for an origin that persists across browsing sessions?
No.

### Do the features in your specification expose information about the underlying platform to origins?
The user agent may select a different cross-origin isolation mode (between logical and concrete), depending on its ability to safely isolate the document.
This depends on its underlying process allocation. While not mandated in the spec, we would recommend user agents make this choice per-platform choice and apply it to all origins equally.

### Does this specification allow an origin to send data to the underlying platform?
No.

### Do features in this specification enable access to device sensors?
No.

### Do features in this specification enable new script execution/loading mechanisms?
No.

### Do features in this specification allow an origin to access other devices?
No.

### Do features in this specification allow an origin some measure of control over a user agent’s native UI?
No.

### What temporary identifiers do the features in this specification create or expose to the web?
None

### How does this specification distinguish between behavior in first-party and third-party contexts?
This API only applies at document load time based on a header and cannot be used by a third-party.

### How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?
No difference with regular browsing mode. No data written to disk.

### Does this specification have both "Security Considerations" and "Privacy Considerations" sections?
Yes.

### Do features in your specification enable origins to downgrade default security protections?
Document-Isolation-Policy enables cross-origin isolation, which allows access to potentially dangerous APIs like SharedArrayBuffers. This is mitigated by allowing the user agent to place the cross-origin isolated document in an appropriately isolated process.
While process allocation is an implementation choice of user agents and cannot be mandated in spec, the spec authors highly recommand that user agents isolate cross-origin isolated documents in separate processes.

### How does your feature handle non-"fully active" documents?
This API only applies at document load time based on a header and thus has no impact on non-"fully active" documents.

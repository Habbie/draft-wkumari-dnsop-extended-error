**Important:** Read CONTRIBUTING.md before submitting feedback or contributing
```




Network Working Group                                          W. Kumari
Internet-Draft                                                    Google
Intended status: Informational                                   E. Hunt
Expires: January 18, 2018                                            ISC
                                                               R. Arends
                                                                 Nominet
                                                             W. Hardaker
                                                                 USC/ISI
                                                             D. Lawrence
                                                     Akamai Technologies
                                                           July 17, 2017


                          Extended DNS Errors
                 draft-wkumari-dnsop-extended-error-02

Abstract

   This document defines an extensible method to return additional
   information about the cause of DNS errors.  The primary use case is
   to extend SERVFAIL to provide additional information about the cause
   of DNS and DNSSEC failures.

   [ Open question: The document currently defines a registry for
   errors.  It has also been suggested that the option also carry human
   readable (text) messages, to allow the server admin to provide
   additional debugging information (e.g: "example.com pointed their NS
   at us.  No idea why...", "We don't provide recursive DNS to
   192.0.2.0.  Please stop asking...", "Have you tried Acme Anvil and
   DNS?  We do DNS right..." (!).  Please let us know if you think text
   is needed, or if a 16bit FCFS registry is expressive enough. ]

   [ Open question: This document discusses extended *errors*, but it
   has been suggested that this could be used to also annotate *non-
   error* messages.  The authors do not think that this is a good idea,
   but could be persuaded otherwise. ]

Status of This Memo

   This Internet-Draft is submitted in full conformance with the
   provisions of BCP 78 and BCP 79.

   Internet-Drafts are working documents of the Internet Engineering
   Task Force (IETF).  Note that other groups may also distribute
   working documents as Internet-Drafts.  The list of current Internet-
   Drafts is at http://datatracker.ietf.org/drafts/current/.





Kumari, et al.          Expires January 18, 2018                [Page 1]

Internet-Draft     draft-wkumari-dnsop-extended-error          July 2017


   Internet-Drafts are draft documents valid for a maximum of six months
   and may be updated, replaced, or obsoleted by other documents at any
   time.  It is inappropriate to use Internet-Drafts as reference
   material or to cite them other than as "work in progress."

   This Internet-Draft will expire on January 18, 2018.

Copyright Notice

   Copyright (c) 2017 IETF Trust and the persons identified as the
   document authors.  All rights reserved.

   This document is subject to BCP 78 and the IETF Trust's Legal
   Provisions Relating to IETF Documents
   (http://trustee.ietf.org/license-info) in effect on the date of
   publication of this document.  Please review these documents
   carefully, as they describe your rights and restrictions with respect
   to this document.  Code Components extracted from this document must
   include Simplified BSD License text as described in Section 4.e of
   the Trust Legal Provisions and are provided without warranty as
   described in the Simplified BSD License.

Table of Contents

   1.  Introduction and background . . . . . . . . . . . . . . . . .   3
     1.1.  Requirements notation . . . . . . . . . . . . . . . . . .   3
   2.  Extended Error EDNS0 option format  . . . . . . . . . . . . .   3
   3.  Use of the Extended DNS Error option  . . . . . . . . . . . .   4
   4.  Defined Extended DNS Errors . . . . . . . . . . . . . . . . .   5
     4.1.  Extended DNS Error Code 1 - DNSSEC Bogus  . . . . . . . .   5
     4.2.  Extended DNS Error Code 2 - DNSSEC Indeterminite  . . . .   5
     4.3.  Extended DNS Error Code 3 - Lame  . . . . . . . . . . . .   5
     4.4.  Extended DNS Error Code 4 - Prohibited  . . . . . . . . .   5
     4.5.  Extended DNS Error Code 5 - TooBusy . . . . . . . . . . .   6
   5.  IANA Considerations . . . . . . . . . . . . . . . . . . . . .   6
   6.  Open questions  . . . . . . . . . . . . . . . . . . . . . . .   7
   7.  Security Considerations . . . . . . . . . . . . . . . . . . .   7
   8.  Acknowledgements  . . . . . . . . . . . . . . . . . . . . . .   7
   9.  References  . . . . . . . . . . . . . . . . . . . . . . . . .   7
     9.1.  Normative References  . . . . . . . . . . . . . . . . . .   7
     9.2.  Informative References  . . . . . . . . . . . . . . . . .   8
   Appendix A.  Changes / Author Notes.  . . . . . . . . . . . . . .   8
   Authors' Addresses  . . . . . . . . . . . . . . . . . . . . . . .   8








Kumari, et al.          Expires January 18, 2018                [Page 2]

Internet-Draft     draft-wkumari-dnsop-extended-error          July 2017


1.  Introduction and background

   There are many reasons that a DNS query may fail, some of them
   transient, some permanent; some can be resolved by querying another
   server, some are likely best handled by stopping resolution.
   Unfortunately, the error signals that a DNS server can return are
   very limited, and are not very expressive.  This means that
   applications and resolvers often have to "guess" at what the issue is
   - e.g the answer was marked REFUSED because of a lame delegation, or
   because there is a lame delegation or because the nameserver is still
   starting up and loading zones?  Is a SERVFAIL a DNSSEC validation
   issue, or is the nameserver experiencing a bad hair day?

   A good example of issues that would benefit by additional error
   information is an error caused by a DNSSEC validation issue.  When a
   stub resolver queries a DNSSEC bogus name (using a validating
   resolver), the stub resolver receives only a SERVFAIL in response.
   Unfortunately, SERVFAIL is used to signal many sorts of DNS errors,
   and so the stub resolver simply asks the next configured DNS
   resolver.  The result of trying the next resolver is one of two
   outcomes: either the next resolver also validates, a SERVFAIL is
   returned again, and the user gets an (largely) incomprehensible error
   message; or the next resolver is not a validating resolver, and the
   user is returned a potentially harmful result.

   This document specifies a mechanism to extend (or annotate) DNS
   errors to provide additional information about the cause of the
   error.  This information can be used by the resolver to make a
   decision regarding whether or not to retry, or by technical users
   attempting to debug issues.

   Here is a reference to an "external" (non-RFC / draft) thing:
   ([IANA.AS_Numbers]).  And this is a link to an
   ID:[I-D.ietf-sidr-iana-objects].

1.1.  Requirements notation

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
   document are to be interpreted as described in [RFC2119].

2.  Extended Error EDNS0 option format

   This draft uses an EDNS0 ([RFC2671]) option to include extended error
   (ExtError) information in DNS messages.  The option is structured as
   follows:





Kumari, et al.          Expires January 18, 2018                [Page 3]

Internet-Draft     draft-wkumari-dnsop-extended-error          July 2017


                                                1   1   1   1   1   1
        0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
   0: |                            OPTION-CODE                        |
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
   2: |                           OPTION-LENGTH                       |
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
   4: | R |                         FLAGS                             |
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
   6: |                             CODE                              |
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

   o  OPTION-CODE, 2 octets (defined in [RFC6891]), for ExtError is TBD.

   o  OPTION-LENGTH, 2 octets ((defined in [RFC6891]) contains the
      length of the payload (everything after OPTION-LENGTH) in octets
      and should be 4.

   o  FLAGS, 2 octets.

   o  CODE, 2 octets.

   Currently the only defined flag is the R flag.

   R - Retry  The R (or Retry) flag provides a hint to the receiver if
      it should retry the query, possibly by querying another server.
      If the R bit is set (1), the sender believes that retrying the
      query may provide a successful answer next time; if the R bit is
      clear (0), the sender believes that it should not ask another
      server.

   The remaining bits in the flags field MUST be set to 0 by the sender
   and SHOULD be ignored by the receiver.

   Code: A code point into the IANA "Extended DNS Errors" registry.

3.  Use of the Extended DNS Error option

   The Extended DNS Error (EDE) is an EDNS option.  It can be included
   in any error response (SERVFAIL, NXDOMAIN, REFUSED, etc) to a query
   that includes an EDNS option.  This document includes a set of
   initial codepoints (and requests to the IANA to add them to the
   registry), but is extensible via the IANA registry to allow
   additional error codes to be defined in the future.

   The R (Retry) flag provides a hint (or suggestion) as to what the
   receiver may want to do with this annotated error.  The mechanism is
   specifically designed to be extensible, and so implementations may



Kumari, et al.          Expires January 18, 2018                [Page 4]

Internet-Draft     draft-wkumari-dnsop-extended-error          July 2017


   receive EDE codes that it does not understand.  The R flag allows
   implementations to make a decision as to what to do if it receives a
   response with an unknown code - retry or drop the query.  Note that
   this flag is only a suggestion or hint.  Receivers can choose to
   ignore this hint.

4.  Defined Extended DNS Errors

   This document defines some initial EDE codes.  The mechanism is
   intended to be extensible, and additional codepoints will be
   registered in the "Extended DNS Errors" registry.  This document
   provides suggestions for the R flag, but the originating server may
   ignore these recommendations if it knows better.

4.1.  Extended DNS Error Code 1 - DNSSEC Bogus

   The resolver attempted to perform DNSSEC validation, but validation
   ended in the Bogus state.  The R flag should be set.

4.2.  Extended DNS Error Code 2 - DNSSEC Indeterminite

   The resolver attempted to perform DNSSEC validation, but validation
   ended in the Indeterminate state.

   Usually attached to SERVFAIL messages.  The R flag should be set.

4.3.  Extended DNS Error Code 3 - Lame

   An authoritative resolver that receives a query (with the RD bit
   clear) for a domain for which it is not authoritative SHOULD include
   this EDE code in the REFUSED response.

   Implementations should not set the R flag in this case (another
   nameserver might not be lame).

4.4.  Extended DNS Error Code 4 - Prohibited

   An authoritative or recursive resolver that receives a query from an
   "unauthorized" client can annotate its REFUSED message with this
   code.  Examples of "unauthorized" clients are recursive queries from
   IP addresses outside the network, blacklisted IP addresses, etc.

   Implementations SHOULD allow operators to define what to set the R
   flag to in this case.







Kumari, et al.          Expires January 18, 2018                [Page 5]

Internet-Draft     draft-wkumari-dnsop-extended-error          July 2017


4.5.  Extended DNS Error Code 5 - TooBusy

   [ Ed: This might be a bad idea.  It is intended to allow servers
   under a DoS (for example a random subdomain attack) to signal to
   recursive clients that they are being abusive and should back off.
   This may be a bad idea -- it may "complete the attack", it may be
   spoofable (by anyone who could also do a MITM style attack), etc.  ]

   A nameserver which is under excessive load (for example, because it
   is experiencing a DoS) may annotate any answer with this code.

   It is RECOMMENDED that implementations set the R flag in this case,
   but may allow operators to define what to set the R flag to.

   [ agreed: bad idea -wjh ]

5.  IANA Considerations

   [This section under construction, beware. ]

   This document defines a new EDNS(0) option, entitled "Extended DNS
   Error", assigned a value of TBD1 from the "DNS EDNS0 Option Codes
   (OPT)" registry [to be removed upon publication:
   [http://www.iana.org/assignments/dns-parameters/dns-
   parameters.xhtml#dns-parameters-11]

   Value  Name                 Status    Reference
   -----  ----------------     ------    ------------------
    TBD   Extended DNS Error    TBD       [ This document ]

   Data Tag Name Length Meaning ---- ---- ------ ------- TBD1 FooBar N
   FooBar server

   The IANA is requested to create and maintain the "Extended DNS Error
   codes" registry.  The codepoint space is broken into 3 ranges:

   o  1 - 16384: Specification required.

   o  16385 - 65000: First Come First Served

   o  65000 - 65534: Experimental / Private use

   The codepoints 0, 65535 are reserved.








Kumari, et al.          Expires January 18, 2018                [Page 6]

Internet-Draft     draft-wkumari-dnsop-extended-error          July 2017


6.  Open questions

   1  Can this be included in *any* response or only responses to
      requests that included an EDNS option?  Resolvers are supposed to
      ignore additional.  EDNS capable ones are supposed to simply
      ignore unknown options.  I know the spec says you can only include
      EDNS0 in a response if in a request -- it is time to reevaluate
      this?

   2  Can this be applied to *any* response, or only error responses?

   3  Should textual information be allowed as well?  What if the only
      thing allowed is a domain name, e.g to point at where validation
      began failing?

7.  Security Considerations

   DNSSEC is being deployed - unfortunately a significant number of
   clients (~11% according to [GeoffValidation]), when receiving a
   SERVFAIL from a validating resolver because of a DNSSEC validaion
   issue simply ask the next (non-validating) resolver in their list,
   and do don't get any of the protections which DNSSEC should provide.
   This is very similar to a kid asking his mother if he can have
   another cookie.  When the mother says "No, it will ruin your
   dinner!", going off and asking his (more permissive) father and
   getting a "Yes, sure, cookie!".

8.  Acknowledgements

   The authors wish to thank Geoff Huston.  They also vaguely remember
   discussing this with a number of people over the years, but have
   forgotten who all they were -- if you were one of them, and are not
   listed, please let us know and we'll acknowledge you.

   I also want to thank the band "Infected Mushroom" for providing a
   good background soundtrack (and to see if I can get away with this!)
   Another author would like to thank the band "Mushroom Infectors".
   This was funny at the time we wrote it, but I cannot remember why...

9.  References

9.1.  Normative References

   [IANA.AS_Numbers]
              IANA, "Autonomous System (AS) Numbers",
              <http://www.iana.org/assignments/as-numbers>.





Kumari, et al.          Expires January 18, 2018                [Page 7]

Internet-Draft     draft-wkumari-dnsop-extended-error          July 2017


   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/
              RFC2119, March 1997,
              <http://www.rfc-editor.org/info/rfc2119>.

9.2.  Informative References

   [GeoffValidation]
              IANA, "A quick review of DNSSEC Validation in today's
              Internet", June 2016, <http://www.potaroo.net/
              presentations/2016-06-27-dnssec.pdf>.

   [I-D.ietf-sidr-iana-objects]
              Manderson, T., Vegoda, L., and S. Kent, "RPKI Objects
              issued by IANA", draft-ietf-sidr-iana-objects-03 (work in
              progress), May 2011.

Appendix A.  Changes / Author Notes.

   [RFC Editor: Please remove this section before publication ]

   From -02 to -03:

   o  Added David Lawrence -- I somehow missed that in last version.

   From -00 to -01;

   o  Fixed up some of the text, minor clarifications.

Authors' Addresses

   Warren Kumari
   Google
   1600 Amphitheatre Parkway
   Mountain View, CA  94043
   US

   Email: warren@kumari.net


   Evan Hunt
   ISC
   950 Charter St
   Redwood City, CA  94063
   US

   Email: each@isc.org




Kumari, et al.          Expires January 18, 2018                [Page 8]

Internet-Draft     draft-wkumari-dnsop-extended-error          July 2017


   Roy Arends
   Nominet
   UK

   Email: TBD


   Wes Hardaker
   USC/ISI
   P.O. Box 382
   Davis, VA  95617
   US


   David C Lawrence
   Akamai Technologies
   150 Broadway
   Cambridge, MA  02142-1054
   US

   Email: tale@akamai.com






























Kumari, et al.          Expires January 18, 2018                [Page 9]
```

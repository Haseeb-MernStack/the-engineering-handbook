# Licensing Explained

> A license is the legal permission that makes code usable by others, and choosing or complying with one is an engineering responsibility, not an afterthought.

**Educational notice.** This page is educational material, not legal advice. License terms are legal text, and their interpretation can depend on jurisdiction and facts. For decisions with real consequences, such as shipping a product, accepting investment, or handling a dispute, consult a qualified lawyer and read the actual license text.

## Overview

By default, code you write is protected by copyright in most countries. That means nobody else may copy, modify, or distribute it without your permission. Publishing code on a public website does not grant that permission. A license is how the copyright holder grants it, and under what conditions.

This matters to every engineer, not only maintainers. When you add a dependency you accept its license. When you copy a snippet you take on its terms. When your company ships software, its obligations flow from the licenses of everything inside it. Ignoring licensing is a common way for engineering teams to create legal and commercial risk without noticing.

The term "open source" has a specific meaning. The Open Source Initiative maintains the Open Source Definition and approves licenses that meet it. Licenses that restrict commercial use, or that restrict use by field of endeavor, do not meet that definition even if the source is visible. These are often called "source-available" licenses.

## Core Concepts

### Two main families

Open source licenses are commonly grouped into permissive and copyleft families. The grouping is a simplification, but it is useful.

| Property | Permissive | Copyleft |
|---|---|---|
| Use, modify, distribute | Allowed | Allowed |
| Keep notices and attribution | Required | Required |
| Must derivative works be released under the same license | No | Yes, under defined conditions |
| Can be included in proprietary software | Generally yes | Depends on the license and how it is combined |
| Examples | MIT, BSD (2-clause and 3-clause), Apache License 2.0, ISC | GNU GPL, GNU LGPL, GNU AGPL, Mozilla Public License 2.0 |

### Permissive licenses

Permissive licenses impose few conditions, mainly that you keep the copyright notice and license text.

- **MIT** and **ISC** are short and simple. They contain a permission grant, a notice requirement, and a disclaimer of warranty.
- **BSD** licenses are similar. The 3-clause version adds a clause restricting use of the authors' names to endorse derived products.
- **Apache License 2.0** is longer. It includes an explicit patent license from contributors and a provision that terminates that patent license for anyone who sues over patents in the work. It also asks that you state significant changes and preserve a NOTICE file where one exists.

### Copyleft licenses

Copyleft uses copyright to require that certain derivative works stay open under the same terms. Strength varies.

- **GNU GPL** is strong copyleft. If you distribute a work based on GPL code, you must make the corresponding source available under the GPL. The conditions are triggered by distribution. Using GPL software internally, without distributing it, generally does not trigger the source-sharing obligation.
- **GNU LGPL** is weaker. It is designed so that applications can link to the library without themselves becoming LGPL, while changes to the library itself must be shared. Details depend on how linking is done and the version of the license.
- **GNU AGPL** extends the GPL to cover users who interact with the software over a network. The intent is to close the gap where a hosted service modifies a program but never "distributes" it.
- **Mozilla Public License 2.0** is file-level copyleft. Modified MPL-covered files must stay under the MPL, but they can be combined with files under other licenses in a larger work.

GPL versions matter. GPL version 2 and version 3 are different licenses with different terms, and compatibility between them and with other licenses is not automatic. Whether a particular combination of licenses is compatible is exactly the kind of question where you need the license texts, and sometimes a lawyer.

### Other things you will encounter

- **Public domain dedications**, such as CC0, aim to waive rights as far as the law allows. Their effect varies by jurisdiction.
- **Creative Commons licenses** are intended for creative works such as documentation and images. The Creative Commons organization advises against using them for software.
- **Source-available licenses**, such as those that restrict competing commercial use, are not open source by the Open Source Definition. Some projects have moved from open source to such licenses. This is a documented pattern, and it affects anyone building on those projects.
- **No license at all** means default copyright applies. You do not have permission to reuse the code.

### How to choose a license for your project

Ask these questions in order:

1. **Do you want maximum adoption, including in proprietary products?** A permissive license such as MIT or Apache 2.0 fits.
2. **Do patents matter to your users or contributors?** Apache 2.0 has an explicit patent grant. MIT is silent on patents.
3. **Do you want improvements to stay open?** Consider copyleft. For libraries, weigh LGPL or MPL against GPL. For network services, weigh AGPL.
4. **Does your ecosystem have a strong convention?** Following it reduces friction for users. Check what the major dependencies in your ecosystem use.
5. **Might you want to change the license later?** Changing license typically requires agreement from all copyright holders. Decide early how contributions are licensed, for example with a DCO or a CLA.

A license choice is hard to reverse once others have contributed. Choose deliberately.

### Compliance basics for consumers

When you use open source in a product or service:

- **Inventory** your dependencies, including transitive ones. A software bill of materials (SBOM) is a structured way to record this.
- **Identify** the license of each component. Machine-readable identifiers from the SPDX list make this tractable.
- **Meet notice obligations.** Most licenses require preserving copyright notices and license text, often in a documentation file or an about screen.
- **Flag copyleft** components and understand how they are combined with your code and whether you distribute them.
- **Keep records** of decisions, and automate license scanning in continuous integration.
- **Set a policy** that lists approved, restricted, and prohibited licenses for your organization.

### Marking your own project

Put a `LICENSE` file at the repository root with the full text. Add an SPDX identifier in package metadata and, if you choose, in source file headers, for example `SPDX-License-Identifier: MIT`. Keep copyright notices accurate.

## Trade-offs

| Choice | What it gives you | What it costs |
|---|---|---|
| Permissive license | Widest adoption; easy for companies to use | Others may use your work in closed products and give nothing back |
| Strong copyleft (GPL) | Derivatives stay open | Many companies avoid it; limits some commercial uses and combinations |
| Weak copyleft (LGPL, MPL) | Middle ground: library changes shared, wider use allowed | Subtler rules; harder to explain and audit |
| Network copyleft (AGPL) | Covers hosted services | Many organizations prohibit it outright; reduces adoption |
| Source-available license | Restricts competitors | Not open source; may reduce community trust and contribution |
| Dual licensing | Possible commercial revenue | Requires owning or aggregating copyright, which makes contributor agreements essential |
| CLA for contributors | Flexibility to relicense | Friction for contributors; some decline to sign |
| DCO sign-off | Light, simple provenance statement | Does not by itself grant relicensing rights |

## Failure Modes

**No license.** A developer publishes useful code without a license and is surprised that companies will not touch it. Prevention: always add a license file.

**Copying code without checking terms.** A snippet from a forum or a repository is pasted into a product. Its license, or lack of one, was never examined. Prevention: treat snippets as dependencies. Check the source and the terms. Note that sites may impose their own licenses on content posted to them.

**Accidental copyleft contamination.** A GPL component is statically linked into a distributed proprietary application and the obligation to release source is noticed only during due diligence. Prevention: automated scanning, an approved-license policy, and review before adding dependencies.

**Ignoring notice requirements.** Teams comply with the headline condition but forget to include attribution files. This is common and easily avoided. Prevention: generate third-party notices as part of the build.

**Incompatible combinations.** Two components under licenses that cannot be combined are shipped together. Prevention: check compatibility before integrating, and ask counsel when it is unclear.

**License change surprises.** A dependency changes license in a new version. Teams upgrade without reading release notes. Prevention: monitor license metadata on upgrade, and pin versions with review.

**Machine-generated code of uncertain origin.** AI assistants can produce code that resembles licensed material. The legal status of such output is unsettled and varies by jurisdiction. Prevention: follow your organization's policy, use available filters, review suspicious large verbatim passages, and prefer to understand and write critical code yourself.

**Contributor license ambiguity.** A project accepts contributions with no stated terms and later cannot relicense or resolve a dispute. Prevention: state contribution terms in `CONTRIBUTING`, and use a DCO or CLA when relevant.

## In Practice

### A starter policy for a small team

- Approved by default: MIT, BSD, ISC, Apache 2.0.
- Requires review: MPL, LGPL, and anything with a custom or unusual license.
- Requires legal approval: GPL, AGPL, source-available licenses, and anything without a license.
- All dependencies recorded in an SBOM, regenerated on each release.
- License scan runs in continuous integration and blocks unapproved additions.

Treat this as an example to adapt with advice, not as a ready-made policy.

### Questions to ask before adding a dependency

- What is its license, and is that identifier accurate for the whole package?
- Do its own dependencies carry different licenses?
- Is the project maintained, and who holds the rights to it?
- Are we distributing it, or only using it internally or as a service?

### Related material

- Supply chain and dependency risk: [Open Source Security](./open-source-security.md) and [Security](../10-security/README.md)
- Contribution terms and provenance: [Contributing to Open Source](./contributing-to-open-source.md)
- Project governance and relicensing decisions: [Community and Governance](./community-and-governance.md)

## Further Reading

- Open Source Initiative, *The Open Source Definition*, opensource.org.
- SPDX License List and specification, spdx.org.
- Free Software Foundation, GNU licenses and their published FAQs, gnu.org.
- Andrew M. St. Laurent, *Understanding Open Source and Free Software Licensing*. A book-length treatment of license families.
- Choose a License (choosealicense.com), a project that summarizes common licenses in plain language.
- The text of each license itself. Short licenses such as MIT can be read in minutes.

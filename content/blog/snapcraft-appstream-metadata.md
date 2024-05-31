+++
title = 'Snapcraft: Adopting appstream metadata'
date = 2024-05-31T15:03:02+05:30
tags = ['snapcraft', 'python', 'appstream', 'linux', 'snaps']
+++

### Introduction

Whatever takes time takes for good. Yeah, so, about on March I created a PR on snapcraft by canonical. It was about adopting more metadata from the parsed appstrean metadata file. The new fields that were made to parse were

1. License
2. Contact
3. Issues
4. Source Code (VCS Browser)
5. Website
6. Donation Link

### What does this change means?

#### For publishers/snapcrafters

Publishers and snapcrafters who also maintains an appstrean metadata for their app, you don't need to maintain the metadata in your snap package separately. Just add the metadata file in your snap and you're good to go. (Also please keep in mind to enable the update metadata from snapcraft option in case you disabled it).

#### For users

You'll not see a change in front of yourself out of the box. But eventually, you'll see all those who have added their appstream metadata to parse from, those pages will get populated with the links and the license automatically. Thus posts like [this](https://forum.snapcraft.io/t/reporting-bugs-in-snaps) or [this](https://discourse.ubuntu.com/t/snap-support) might eventually reduce in the Snapcraft forum or Ubuntu discourse.

### How does that work?

Well as snapcraft did with summary, description, title etc, here again we're using `lxml.etree` library to scrap the `xml` file. The finding these fields.

1. `project_license` for License
2. `update_contact` for contact
3. `bugtracker` for issues
4. `donation` for Donation Links
5. `homepage` for Website Links
6. `vcs-browser` for Source Code

But, there is a twist, the `links` mentioned above were previously `Optional[str]`, [but this new PR sets this as `Optional[List[str]]`](https://github.com/canonical/snapcraft/blob/main/snapcraft/meta/extracted_metadata.py#L54-L67).

```python
    contact: Optional[List[str]] = None
    """The extracted package contact"""

    donation: Optional[List[str]] = None
    """The extracted package donation"""

    issues: Optional[List[str]] = None
    """The extracted package issues"""

    source_code: Optional[List[str]] = None
    """The extracted package source code"""

    website: Optional[List[str]] = None
    """The extracted package website"""
```

What does that mean? Well, it means that you can have multiple links for the same field. For example, you can have multiple `issues` links, or multiple `source code` links. This is a good feature as it allows the publisher to give users multiple options to contact, raise a bug report or just to browse the source-code. But, this also means that the existing manifests might break. So, for that, we came up with [`pydantic.validator`](https://docs.pydantic.dev/latest/concepts/validators/) to validate the existing manifests. [This validator will check if the links are of type `str` and if they are, it will convert them to `UniqueStrList`](https://github.com/canonical/snapcraft/blob/main/snapcraft/models/project.py#L861-L868).

```python
    @pydantic.validator(
        "contact", "donation", "issues", "source_code", "website", pre=True
    )
    @classmethod
    def _validate_urls(cls, field_value):
        if isinstance(field_value, str):
            field_value = cast(UniqueStrList, [field_value])
        return field_value
```

### So, what's next?

#### My plans:

1. Try to look into a way, so that screenshot can also be parsed from the appstream metadata.
2. Try to raise a issue/PR in the [new flutter based App Center](https://github.com/ubuntu/app-center) to show the links in the app details page more prominently.
3. Try to raise an issue in the snapcraft forum, about adding these links mandatory for all snaps.

### Acknowledgements

Thanks a lot to everyone who helped me in this PR. Especially to

- [Claudio Matsuoka](https://github.com/cmatsuoka)
- [Sergio Schvezov](https://github.com/sergiusens)
- [Callahan John Kovacs](https://github.com/mr-cal)
- [Alex Lowe](https://github.com/lengau)
- [Tiago Nobrega](https://github.com/tigarmo)

### References

1. [canonical/snapcraft #4562](https://github.com/canonical/snapcraft/pull/4652)
2. [Snapcraft Docs](https://snapcraft.io/docs/snapcraft-top-level-metadata)

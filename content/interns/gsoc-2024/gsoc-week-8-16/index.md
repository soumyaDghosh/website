+++
title = 'GSOC: Week 8 to Week 16'
date = 2024-10-01T18:00:00+05:30
images = ['gsoc.png']
keywords = ['KDE', 'Discover', 'GSOC', '2024', 'Snap', 'KCM']
tags = ['KDE', 'GSOC', '2024', 'Snap']
+++

The End is here! 

The long journey of GSoC (which got streched into 16 weeks) is nearing to an end. And in this blog, I will share a few of the things I have done, and some of the things I have left to do. Let's start with the main works.

## Snap KCM! It's here!

I have finally succeeded to create a KCM for snaps. This is written using C++, Qt (Qml), Kirigami, Snapd-Glib Api. The flow is something like this

### Flowchart of the Snap KCM

SnapBackend  &rarr; Collects all snaps &rarr; Collects all the connections &rarr; sort those, map all connections with their respective snaps &rarr; saves those to a custom class(KCMSnap) &rarr; Lists them via QML

The architecture of the KCMSnap class is like these

### KCMSnap class architecture


|   |
|---|
| QObject |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&darr; |
| KCMSnap |

#### Properties: 
```
- m_snap (QSnapdSnap)
- m_plugs (QList<QSnapdPlug>)
- m_slots (QList<QSnapdSlot>)
```
#### Methods:

```
- snap => QSnapdSnap
- plugs => QList<QSnapdPlug>
- slots => QList<QSnapdSlot>
- icon => QVariant
```

#### QProperties:
```
- QSnapdSnap *snap READ snap CONSTANT
- QList<QSnapdPlug *> plugs READ plugs CONSTANT
- QList<QSnapdSlot *> slots READ slots CONSTANT
- QVariant icon READ icon CONSTANT
```

And here is an image of it!!!

![](https://invent.kde.org/soumyadghosh/snap-kcm/-/raw/master/resources/plugs_permissions_page.png)

Doesn't this look great? :wink:

And to top this off, I made a snap for this. One can find more about it in the snap forum[post](https://forum.snapcraft.io/t/request-for-manual-review-for-snap-kcm/43252).


## Snap Maintenances

### Neochat Snap

Upgraded the snap to the latest release. Needed to build some parts from source.

https://invent.kde.org/network/neochat/-/merge_requests/1923

### Snapcraft Desktop Integration

Working into improving the launch speed for KDE Snaps.

https://invent.kde.org/soumyadghosh/snapcraft-desktop-integration

## Discover

### Fetching Icons of Snaps

I reused the logic of fetching icons for snaps, which I used in Snap KCM. The logic is something like this

1. QSnapdSnap &rarr; icon
2. QSnapdSnap &rarr; media &rarr; `mediaType == icon`
3. QSnapdClient &rarr; getIcon &rarr; icon

https://invent.kde.org/plasma/discover/-/merge_requests/936

And is this it? End of my journey with KDE? Well, no. It just started. And I have a long way to go. With Snaps, With Carl Schwan's KDE Apps Initiative. I want to make some small, handy utilites for KDE.

Thank you everyone, for helping me out!!

- [Scarlett Moore](https://invent.kde.org/scarlettmoore)
- [Aleix Pol](https://invent.kde.org/apol)
- [Nick Logozzo](https://github.com/nlogozzo)
- [Joshua Goins](https://invent.kde.org/redstrate)

A big thanks to the KDE Development Matrix channel members for all the help. I am really grateful to you!

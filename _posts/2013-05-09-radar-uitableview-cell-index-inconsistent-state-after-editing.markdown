---
layout: post
title: "Radar: UITableView cell index inconsistent state after editing"
date: 2013-05-09 08:59
comments: true
categories: [iOS, radar, bugreport]
---

When working on `UITableView` I came across a bug in iOS, and have just got around
to filing my first ever apple bug report. Under certain conditions it's possible
to get the index paths of cells in the table to become inconsistent with the content.

Filed as rdar://13846681 and on [OpenRadar](http://openradar.appspot.com/13846681).

### Summary

It's possible to get a `UITableView` with differing cell heights into a state
where the cell index paths are inconsistent with the datasource.

<!-- more -->

### Steps to Reproduce

1. Create a `UITableView` in which the topmost cell is over twice the height of
the one immediately below it.
2. Enable editing (specifically we want row reordering).
3. Enter edit mode and scroll the table down so that you can only just see the
reordering grabber on the top cell.
4. Tap the grabber. This will switch cells 0 and 1 (due to the height difference).

The problem occurs only when the row switch causes a row to disappear off screen
(hence why we need the top row to be over double the height of the one below it).

### Expected Results

The `tableView:moveRowAtIndexPath:toIndexPath:` datasource method should be called
with the correct index paths, and the index paths of the cells in the table should
be updated appropriately.

### Actual Results

The datasource method is called correctly, but the cells in the table are left with
inconsistent index paths.

### Regression

Tested in iOS 4, 5 & 6 with the same results.

### Notes

I have attached a sample project which demonstrates this. To see the effects:

1. Open the project
2. Click the edit button
3. Scroll the table down so that the grabber icon is only just visible for the
   top cell (cell 0).
4. Single tap the reordering grabber.
 
The table now has an inconsistent index path state. The log status button shows
the index path (row) for each of the cells currently visible table, and the
corresponding index of the content of that cell in the backing array. These should
always match. Pressing it after the above actions will reveal that this isn't the
case, and if you scroll back to the top of the table and log the status again
you'll see that some of the cell content is repeated. Further scrolling of the
table will cause blank rows to appear etc.

## Sample project

The sample project which I uploaded is available on GitHub at
[github.com/sammyd/UITableView-RadarSample](https://github.com/sammyd/UITableView-RadarSample).
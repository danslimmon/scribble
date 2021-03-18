# scribble

Scribble is a simple database built on top of Git. You can use it in a distributed fashion, e.g.
with a GitHub repo. Changes made from different working copies will be eventually coalesced into a
coherent, agreed-upon state.

Scribble can be used to store objects:

```go
import (
    "fmt"
    "github.com/danslimmon/scribble"
)

func main() {
    // New takes a path to an already initialized Git working copy with an origin remote already
    // configured.
    db := scribble.New("/path/to/your/git/repo")

    // Add a record to the database
    db.Write(scribble.Record{
        // Records may be assigned a type, to assist in grouping and querying.
        Type: "note",

        // You can query for all records with a given set of tags.
        Tags: []string{"foo", "bar"},

        // Content can be any object that can by Marshaled and Unmarshaled with the
        // encoding/json package.
        Content: map[string]string{"hello": "world!"},
    })

    // Iterate over all records of type "note"
    db.Each(scribble.Query{Type: "note"}, func(r scribble.Record) error {
        fmt.Printf("Content of note %s: '%v'\n", r.ID(), r.Content)
    }

    // Modify the record with the given ID. This creates a new record and deletes the old record.
    newID, err := db.Alter("bcdef012-3456-789a-bcde-f0123456789a", func(r scribble.Record) (scribble.Record, error) {
        r.Content = struct{X, Y float64}{X: 3.14, Y: 2.72}
        return r, nil
    })

    // Delete the record with the given ID
    db.Delete("abcdef01-2345-6789-abcd-ef0123456789")
}
```

The other thing you can do with Scribble is operate on a tree:

```go
func main() {
    db := scribble.New("/path/to/your/git/repo")

    // TreeHandle takes the name of a tree, which is this example already exists in the database.
    tree := db.TreeHandle("mytree")

    // Iterate (depth-first) over the nodes of the tree. Node labels are strings.
    tree.Walk(func(node scribble.TreeNode) error {
        fmt.Printf(
            "Node with ID '%s' has label '%s' and %d child nodes\n",
            node.ID(),
            node.Label,
            len(node.Children),
        )
        return nil
    )}

    // Add a child node to the root of the tree
    nodeID, _ := tree.AddChild(scribble.RootNodeID, scribble.TreeNode{
        Label: "this node is a child of the root node",
    })

    // Change the node's label
    tree.AlterNode(nodeID, func(node scribble.TreeNode) (scribble.TreeNode, error) {
        node.Label = "check out my cool new label!"
        return node, nil
    })

    // Add a child to the node we just added (this new node will be a grandchild of the root node)
    tree.AddChild(nodeID, scribble.TreeNode{Label: "this node is two levels down from the root node"})

    // Delete the node that we initially created
    tree.DeleteNode(nodeID)
}
```

By default, every change you make with Scribble gets immediately committed to Git and pushed
(asynchronously) to the origin. You may instead choose to batch up your changes into a single commit
`(to do: write an example for this)`

## Consistency and conflicts

For record storage (`DB.Each`, `DB.Delete`, and so on), there is no possibility of a conflict. If
two instances of Scribble concurrently call `DB.Alter` against the same record ID, then the database
just ends up with two distinct copies of that record. If a record is altered by one Scribble
instance and deleted by another, the deletion is ignored.

Tree nodes work similarly. If a tree node's label is concurrently altered by two Scribble
instances, the change with the later wall clock time wins. If a tree node is altered by one instance
and deleted by another instance, the deletion is ignored. If one Scribble instance adds a child to a
tree node, and another instance also adds a child, both children end up in the tree, with their
ordering determined by the wall clock times of the changes.

## what is this?

This file contains content about PLUS blank headers, a feature not defined in the present version of PLUS

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+--------------------------------------------------------------+
|       UDP source port        |      UDP destination port     |
+--------------------------------------------------------------+
|       UDP length             |      UDP checksum             |
+--------------------------------------------------------------+
/        first four bytes MUST NOT equal version / magic       \
\                                                              /
/         transport protocol header/payload (encrypted)        \
\                                                              /
~~~~~~~~~~~~~
{: #fig-header-blank title="PLUS blank header"}

~~~~~~~~~~~~~
    `- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -'
    `                        +============+                       '
    `                       /              \    TO_IDLE           '
  +-----------+----------->(      zero      )<---------+          '
  ^ `         ^             \              /            \         '
  | ` TO_IDLE |              +============+              \        '
  | `          \   blank a->b /           \ basic a->b   /        '
  | `           \            V             V            /         '
  | `            +============+  basic    +============+          '
  | `        +->/              \-------->/              \<-+      '
  | `  blank | (  anon-uniflow  ) a->b  (  plus-uniflow  ) | any  '
  | `   a->b +--\              /         \              /--+ a->b '
  | `            +============+           +============+          '
  | `- - - - - - - - - - - - \ basic b->a /- - - - - \ - - - - - -'
  |                           V          V            \
  | TO_IDLE                  +============+            \
  +<------------------------/              \ wrong CID  \
  |                        (  associating   )------------+
  |                         \              /              \
  | TO_ASSOCIATED            +============+                |
  +<-----------------------+        | basic a->b           |
  |                         \       V                      |
  |                          +============+                |
  |                      +->/              \   wrong CID   |
  |            any a<->b | (   associated   )--------------+
  |                      +--\              /               |
  | TO_ASSOCIATED            +============+                |
  +<-----------------+        | stop y->z                  |
  |                   \       V                            V
  |                    +============+                +============+
  |                +->/              \  wrong CID   /              \
  |      any a<->b | (    stopping    )----------->(    cid-fail    )
  |                +--\              /              \              /
  | TO_STOPWAIT        +============+                +============+
  +------------+        | stop z->y
                \       V
                 +============+
                /              \
               (   stop-wait    )
                \              /
                 +============+
~~~~~~~~~~~~~
{: #fig-states title="Transport-independent state machine as implemented by PLUS"}
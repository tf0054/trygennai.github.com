> genn.aiは、大きくTupleStoreServerとGungnirServerから構成され、
	前者TupleStoreServerは、外部からjson形式のデータをRESTで受け取りStormに渡します(正確には、そのためにKafkaに格納します)。
	後者GungnirServerは、genn.ai独自の **クエリ** で書かれたイベント処理ロジックをコンパイルし、Stormに登録します。
	(この処理ロジックでは、通常、最初にKafkaからデータを読み出します)

> genn.aiでは、このjson形式で受け取るデータを **タプル** (Tuple)と呼び、クエリのコンパイルにより出来上がるものは **トポロジ** (Topology)と呼びます。
	(タプル及びトポロジはStormの用語をそのまま借りています)

> TupleStoreServerが受け取る **タプル** の定義や操作については [DDL](ddl.html)のページを、後者GungnirServerが担当するクエリやトポロジの操作については [DML](dml.html)のページをご参照下さい。

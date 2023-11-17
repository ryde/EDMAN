edman
=====

|py_version|

|  KEK IMSS SBRC/PF Experimental Data Management System.
|  jsonファイル(階層構造になった実験データ等)を親子構造を解析し、MongoDBに投入します。

Requirement
-----------
-   pymongo
-   python-dateutil
-   jmespath

and MongoDB.

Modules Usage
-------------

◯Create

::

    import json
    from edman import DB, Convert

    # Load json into a dictionary
    json_dict = json.load(json_file)

    # json to json for edman
    convert = Convert()
    converted_edman = convert.dict_to_edman(json_dict)

    # insert
    con = {'port':'27017', 'host':'localhost', 'user':'mongodb_user_name', 'password':'monogodb_user_password', 'database':'database_name', 'options':['authSource=auth_database_name']}
    db = DB(con)
    result = db.insert(converted_edman)

◯Read

::

    from pathlib import Path
    from bson import ObjectId
    from edman import DB, JsonManager, Search

    con = {'port':'27017', 'host':'localhost', 'user':'mongodb_user_name', 'password':'monogodb_user_password', 'database':'database_name', 'options':['authSource=auth_database_name']}
    db = DB(con)
    search = Search(db)
    collection = 'target_collection'

    # Same syntax as pymongo's find query
    query = {'_id':ObjectId('OBJECTID')}

    # example, 2 top levels of parents and 3 lower levels of children (ref mode)
    search_result = search.find(collection, query, parent_depth=2, child_depth=3)

    # Save search results
    save_dir = Path('path_to')
    jm = JsonManager()
    jm.save(search_result, save_dir, name='search_result', date=True)
    # saved. path_to/20231116102047106179_search_result.json

◯Update

::

    import json
    from bson import ObjectId
    from edman import DB

    modified_data = json.load(modified_json_file)

    # emb example
    # Same key will be modified, new key will be added
    # modified_data = {"key": "modified value", "child": {"key": "value"}}

    # ref example
    # Same key will be modified, new key will be added
    # modified_data = {"key": "modified value", "new_key": "value"}

    con = {'port':'27017', 'host':'localhost', 'user':'mongodb_user_name', 'password':'monogodb_user_password', 'database':'database_name', 'options':['authSource=auth_database_name']}
    db = DB(con)
    result = db.update(collection, ObjectId('objectid'), modified_data, structure='ref')

◯Delete

::

    from edman import DB

    con = {'port':'27017', 'host':'localhost', 'user':'mongodb_user_name', 'password':'monogodb_user_password', 'database':'database_name', 'options':['authSource=auth_database_name']}
    db = DB(con)
    result = db.delete(objectid, collection, structure='ref')

◯File Upload

::

    from pathlib import Path
    from bson import ObjectId
    from edman import DB, File

    con = {'port':'27017', 'host':'localhost', 'user':'mongodb_user_name', 'password':'monogodb_user_password', 'database':'database_name', 'options':['authSource=auth_database_name']}
    db = DB(con)
    edman_file = File(db.get_db)

    p = Path()
    current = p.cwd()
    upload_path = current / "memo.txt"

    if upload_path.exists():
        file_path = (upload_path,)
        result = edman_file.upload('plate', ObjectId('objectid'), file_path, structure='ref')
        print('upload:', result) # bool


Json Format
-----------
| example

::

    {
        "Beamtime":
        [
            {
                "date": {"#date": "2019-09-17"},
                "expInfo":[
                        {
                            "time": {"#date": "2019/09/17 13:21:45"},
                            "int_value": 135,
                            "float_value":24.98
                        },
                        {
                            "time": {"#date": "2019/09/17 13:29:12"},
                            "string_value": "hello world"
                        }
                ]
            },
            {
                "date": {"#date": "2019-09-18"},
                "expInfo":[
                        {
                            "array_value": ["string", 1234, 56.78, true, null],
                            "Bool": false,
                            "Null type": null
                        }
                ]
            }
        ]
    }

| #date{}で囲むと日付書式がdatetime型に変換されます。書式はdateutilと同等。
|     https://dateutil.readthedocs.io/en/stable/parser.html#module-dateutil.parser
| 使用できる型はjsonに準拠。整数、浮動小数点数、ブール値、null型、配列も使用可。
| jsonのオブジェクト型はEdmanでは階層構造として認識されます。
|
| 予約コレクション名
|   ・他ドキュメントのリファレンスと同じ名前(_ed_parent,_ed_child,_ed_file) ※システム構築時にのみ変更可
| 予約フィールド名
|   ・日付表現の変換に使用(#date) ※システム構築時にのみ変更可
|   ・ObjectIdと同じフィールド名(_id)
|
|  設定変更については configuration-details_.
| その他MongoDBで禁止されているフィールド名は使用不可
|      https://docs.mongodb.com/manual/reference/limits/#naming-restrictions
|
| MongoDBの1つのドキュメントの容量上限は16MBですが、
|     emb形式の場合はObjectId及びファイル追加ごとのリファレンスデータを含むため、16MBより少なくなります。
|     ref形式の場合は1階層につきObjectId、及びroot(一番上の親)以外は親への参照もデフォルトで含め、子要素やファイルが多いほど参照が増えるため16MBより少なくなります。
|
|  ◯emb(Embedded)とref(reference)について
|  embはjsonファイルの構造をそのままドキュメントとしてMongoDBに投入します。
|   ・親子構造を含め全て一つのコレクションに保存します。
|  refはjsonの親子構造を解析し、オブジェクト単位をコレクションとし、親子それぞれをドキュメントとして保存します。
|   ・親子関係はリファレンスによって繋がっているので指定のツリーを呼び出すことができます。


Type Conversion
---------------

|  ◯型変換について(refのみ)
|   ・edman.DB.bson_type()にて値の型変換をコレクション別に一度に行うことができます
|   ・DB内のすべてのコレクションが変換されます
|   ・DBにあってJSONファイルにないキーは無視されます
|   ・型一覧にない型を指定した時はstrに変換します
|   ・型一覧:
|      [int,float,bool,str,datetime]
|
|   ・値がリストの時
|       ・双方どちらかがリストでない時は無視
|       ・JSON側が単一、DB側が複数の時は単一の型で全て変換する
|           JSON:['str']
|           DB:['1','2','3']
|      ・JSON側よりDB側が少ない時はJSON側は切り捨て
|           JSON:['str'、'int', 'int']
|           DB:['1',2]
|      ・JSON側よりDB側が多い時は、リストの最後の型で繰り返す
|           JSON:['str'、'int']
|           DB:['1',2,3,4,5]


| ・型変換用の辞書の例:

::


      {
          "コレクション名":{
              "キー": "変更する型",
              "キー2": "変更する型",
          },
          "コレクション名2":{
              "キー": ["変更する型","変更する型"],
          }
      }

Attached FIle Management
------------------------

|  ◯ドキュメントへのファイル添付について
|   ・DB内のすべてのドキュメントは関連ファイルを添付することができます
|   ・ドキュメント内でのGrid.fsへのリファレンスのデフォルトのキーは「_ed_file」です(JSONファイルには記述されません)
|   ・zipで圧縮してjsonファイルと添付ファイルを一緒に投入することができます
|   ・拡張子は「.zip」のみ。パスワードは利用不可
|   ・jsonファイルの「_ed_attachment」キーにディレクトリ及びファイル名のパスを設定
|   ・ディレクトリはJSONファイルからの相対パスで記述


::


    zip_dir
    ├─dir1
    │ ├─sample_photo.jpg
    │ └─experiment.cbf
    ├─dir2
    │ └─sample_photo2.jpg
    ├─dir3
    │ ├─sample_photo3.jpg
    │ └─memo.txt
    └─tree.json


|  ・上記構造の場合のtree.jsonの内容例


::

    {
    "beamtime":
        {
            "date": {"#date": "2023-11-01 16:00:00"},
            "beamline": "AR-NE3A",
            ""float_data: 234.56,
            "expInfo": {
                "userid": "user1"
                "bool_flg": true,
            },
            "file":[
                    {
                        "int_data": 1234,
                        "_ed_attachment":["dir1/sample_photo.jpg", "dir1/experiment.cbf"]
                    },
                    {
                        "list_data": ["A","B"],
                        "_ed_attachment":["dir2/sample_photo2.jpg"]
                    }
                ],
            "sample": {
                    "date": {"#date": "2023-11-01 16:00:00"},
                    "sampleid": "sample1",
                    "_ed_attachment":["dir3/sample_photo3.jpg", "dir3/memo.txt"]
            }
        }
    }


.. _configuration-details:

Configuration Details
---------------------
|  ◯設定について
|
|  class Config:
|
|      # ドキュメント内でedmanが使用するリファレンス用のキー
|      parent = '_ed_parent'  # 親のリファレンス情報
|      child = '_ed_child'  # 子のリファレンス情報
|      file = '_ed_file'  # Grid.fsのリファレンス情報
|
|      # Grid.fsのデフォルトコレクション名
|      fs_files = 'fs.files'  # ファイルコレクション名
|      fs_chunks = 'fs.chunks'  # ファイルチャンクコレクション名
|
|      # ユーザがJSON内で使用するキー
|      # 日付に変換する場合のキー　例: "startDate": {"#date": "2020-07-01 00:00:00"}
|      date = '#date'
|
|      # JSON内で使用する添付ファイルディレクトリ用のキー
|  　  # 例: "_ed_attachment":["dir1/sample_photo.jpg", "dir1/experiment.cbf"]
|      file_attachment = '_ed_attachment'
|
|  ◯設定の変更について
|   ・edman.Configクラスにてクラス変数を変更することによりシステム稼働時に設定変更が可能です
|   ・通常はデフォルト設定のままで問題ありませんが、特定のキー名をデータ内で利用したい場合のみ変更してください
|   ・キー名「#date」、「_ed_attachment」以外はJSONデータでは使用しないでください


Scripts Usage
-------------

|  コマンドライン用実行スクリプトはedman_cliを利用してください
|  https://github.com/ryde/edman_cli

Install
-------
|  Please install MongoDB in advance.

pip install::

 pip install edman

Licence
-------
MIT

API Document
------------
https://ryde.github.io/edman/

PyPI Project
------------
https://pypi.org/project/edman/

Author
------

[ryde](https://github.com/ryde)

[yuskyamada](https://github.com/yuskyamada)

.. |py_version| image:: https://img.shields.io/badge/python-3.11-blue.svg
    :alt: Use python

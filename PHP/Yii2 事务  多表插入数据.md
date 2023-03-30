### Yii2 使用事务  对多个表插入数据
```
$trans = Yii::$app->db->beginTransaction();//开启事务
$model_1 = new table1;
$model_2 = new table2;
$model_3 = new table3;
try{
    if (!$model1->save()) {
        throw new Exception();
    }else{
        if (!$model2->save()) {
            throw new Exception();
        }else{
            if(!$model3->sace()){
                throw new Exception();
            }
        }
    }
    $trans->commit();
    return $this->success();
}catch(Exception $e){
    $trans->rollBack();
    return $this->error();
}
```


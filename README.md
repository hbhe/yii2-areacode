自带省市区(县)数据库，三级联动演示(Yii2)
===========================

Installation
------------

提供了migration文件, 先执行 php yii migrate/up 生成数据库表。以下为三级联动demo代码, 供参考


Usage
-----

```php
1. Controller文件
class AreaCodeController extends Controller
{
    public function actionIndex()
    {
        $dataProvider = new ActiveDataProvider([
            'query' => AreaCode::find(),
        ]);

        return $this->render('index', [
            'dataProvider' => $dataProvider,
        ]);
    }

    // ...

    // 省变化时获取新的市
    public function actionSubcat()
    {
        Yii::info($_POST);
        $out = [];
        if (isset($_POST['depdrop_all_params'])) {
            $parent_id = $_POST['depdrop_all_params']['parent_id'];
            $selected_id = $_POST['depdrop_all_params']['selected_id'];
            $out = Yii::$app->db->cache(function ($db) use ($parent_id) {
                return AreaCode::find()->select(['id', 'name'])->where(['parent_id' => $parent_id])->asArray()->all();
            }, YII_DEBUG ? 3 : 24 * 3600);
            return \yii\helpers\Json::encode(['output' => $out, 'selected' => $selected_id]);
        }

        return \yii\helpers\Json::encode(['output' => '', 'selected' => '']);
    }

    // 省市变化时获取新的区
    public function actionDistrictSubcat()
    {
        Yii::info($_POST);
        $out = [];
        if (isset($_POST['depdrop_all_params'])) {
            $parent_id = $_POST['depdrop_all_params']['area_id'];
            $selected_id = $_POST['depdrop_all_params']['selected_district_id'];
            $out = Yii::$app->db->cache(function ($db) use ($parent_id) {
                return AreaCode::find()->select(['id', 'name'])->where(['parent_id' => $parent_id])->asArray()->all();
            }, YII_DEBUG ? 3 : 24 * 3600);
            return \yii\helpers\Json::encode(['output' => $out, 'selected' => $selected_id]);
        }

        return \yii\helpers\Json::encode(['output' => '', 'selected' => '']);
    }

}


2. Model文件
class AreaCode extends ActiveRecord
{
    public static function tableName()
    {
        return '{{%area_code}}';
    }

    public static function find()
    {
        return new AreaCodeQuery(get_called_class());
    }

    public function rules()
    {
        return [
            [['type'], 'integer'],
            [['name'], 'string', 'max' => 64],
            [['zip', 'id', 'parent_id'], 'string', 'max' => 16],
        ];
    }

    public function attributeLabels()
    {
        return [
            'id' => 'ID',
            'type' => '层级',
            'name' => '名称',
            'parent_id' => '上级代码',
            'zip' => '邮编',
        ];
    }

    public static function getChinaProvinceAreaCode()
    {
        return AreaCode::find()->where(['parent_id' => 1])->all();
    }

    public static function getProvinceOption()
    {
        $key = [__METHOD__];
        $data = Yii::$app->cache->get($key);
        if ($data !== false) {
            return $data;
        }

        $models = self::getChinaProvinceAreaCode();
        $value = ArrayHelper::map($models, 'id', 'name');
        Yii::$app->cache->set($key, $value, 24 * 3600);
        return $value;
    }
}


3. View文件（多级联动使用了 \kartik\depdrop\DepDrop 挂件）

<div class="my-form">

    <?php $form = ActiveForm::begin(); ?>

        <?php echo $form->field($model, 'area_parent_id')->dropDownList(AreaCode::getProvinceOption(), ['prompt' => '选择省...', 'id' => 'parent_id']); ?>

        <?php echo Html::hiddenInput('selected_id', $model->isNewRecord ? '' : $model->area_id, ['id' => 'selected_id']); ?>
        <?php echo $form->field($model, 'area_id')->widget(\kartik\depdrop\DepDrop::classname(), [
            'options' => ['id' => 'area_id', 'class' => '', 'style' => ''],
            'pluginOptions' => [
                'depends' => ['parent_id'], 
                'placeholder' => '选择市...',
                'initialize' => $model->isNewRecord ? false : true,
                'url' => Url::to(['/area-code/subcat']), // 当省onchange时通过此url获取新的市
                'params' => ['selected_id']
            ],
        ]); ?>

        <?php echo Html::hiddenInput('selected_district_id', $model->isNewRecord ? '' : $model->district_id, ['id' => 'selected_district_id']); ?>
        <?php echo $form->field($model, 'district_id')->widget(\kartik\depdrop\DepDrop::classname(), [
            'options' => ['id' => 'district_id', 'class' => '', 'style' => ''],
            'pluginOptions' => [
                'depends' => ['parent_id', 'area_id'], 
                'placeholder' => '选择区...',
                'initialize' => $model->isNewRecord ? false : true,
                'url' => Url::to(['/area-code/district-subcat']),
                'params' => ['selected_district_id']
            ],
        ]); ?>

    <div class="form-group">
        <?php echo Html::submitButton($model->isNewRecord ? '创建' : '更新', ['class' => $model->isNewRecord ? 'btn btn-success' : 'btn btn-primary']) ?>
    </div>

    <?php ActiveForm::end(); ?>

</div>

```

 ![截图](https://github.com/hbhe/yii2-areacode/blob/master/screenshot.png)
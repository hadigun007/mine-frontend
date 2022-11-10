## Controller 
/Users/tennetdepositoryfullstack1/go/pkg/mod/github.com/!go!admin!group/go-admin@v1.2.23/plugins/admin/controller


- func (h *Handler) [ApiCreate](#api_creatego--func-h-handler-apicreatectx-contextcontext)<!--  -->
- func (h *Handler) ApiCreateForm(ctx *context.Context) <!-- -->
- func (h *handler) ApiDetail(ctx *ctx.Context) <!-- -->
- func (h *Handler) ApiList(ctx *context.Context) <!-- -->
- func (h *Handler) ApiUpdate(ctx *context.Context) <!-- -->
- func (h *Handler) ApiUpdateForm(ctx *context.Context) <!-- -->
- func (h *Handler) Auth(ctx *context.Context) <!-- Auth check the input password and username for authentication. -->
- func (h *Handler) Logout(ctx *context.Context) <!-- Logout delete the cookie. -->
- func (h *Handler) ShowLogin(ctx *context.Context) <!-- ShowLogin show the login page. -->
- func (h *Hanlder) TestInfoUrl(t *testing.T) <!-- -->
- func (h *Handler) TestIsNewUrl(t *testing.T) <!-- -->
- func (h *Handler) Delete(ctx *context.Context)  <!-- -->
- func (h *Handler) ShowDetail(ctx *context.Context) <!-- -->
- func (h *Handler) ShowForm(ctx *context) <!-- -->
- func (h *Handler) EditForm(ctx *context.Context) <!-- -->
- func (h *Handler) GlobalDetaultHandler(ctx *context.Context)  <!-- -->
- func (h *Handler) ShowInstall(ctx *context.Context)  <!-- -->
- func (h *Handler) CheckDatabase(ctx *context.Context)  <!-- -->
- func (h *Handler) ShowMenu(ctx *context.Context)  <!-- -->
- func (h *Handler) ShowNewMenu(ctx *context.Context)  <!-- -->
- func (h *Handler) ShowEditMenu(ctx *context.Context)  <!-- -->
- func (h *Handler) DeleteMenu(ctx *context.Context)  <!-- -->
- func (h *Handler) EditMenu(ctx *context.Context)  <!-- -->
- func (h *Handler) NewMenu(ctx *context.Context)  <!-- -->
- func (h *Handler) MenuOrder(ctx *context.Context)  <!-- -->
- func (h *Handler) ShowNewForm(ctx *context.Context)  <!-- -->
- func (h *Handler) NewForm(ctx *context.Context)  <!-- -->
- func (h *handler) Operation(ctx *context.Context)  <!-- -->
- func (h *Handler) RecordOperationLog(ctx *context.Context)plugin_tmpl.go  <!-- -->
- func (h *Handler) GetPluginsPageJS(data PluginsPageJSData) template.JS   <!-- -->
- func (h *Handler) Plugins(ctx *context.Context)  <!-- -->
- func (h *Handler) PluginStore(ctx *context.Context)  <!-- -->
- func (h *Handler) PluginDetail(ctx *context.Context)  <!-- -->
- func GetPluginBoxParamFromPlug(plug plugin.Plugin) PluginBoxParam  <!-- -->
- func (h *Handler) PluginDownload(ctx *context.Context)  <!-- -->
- func (h *Handler) ServerLogin(ctx *context.Context)  <!-- -->
- func (h *Handler) ShowInfo(ctx *context.Context)  <!-- -->
- func (h *Hanlder) Assets(ctx *context.Context)  <!-- -->
- func (h *Handler) Export(ctx *context.Context)  <!-- -->
- func (h *Handler) SystemInfo(ctx *context.Context)  <!-- -->
- func (h *Handler) Update(ctx *context.Context)  <!-- -->


-----------------------------------
### Controller API

##### api_create.go > func (h *Handler) ApiCreate(ctx *context.Context) 
``` shell
# 
func (h *Handler) ApiCreate(ctx *context.Context) {
	param := guard.GetNewFormParam(ctx)

	if len(param.MultiForm.File) > 0 {
		err := file.GetFileEngine(h.config.FileUploadEngine.Name).Upload(param.MultiForm)
		if err != nil {
			response.Error(ctx, err.Error())
			return
		}
	}

	err := param.Panel.InsertData(param.Value())
	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.Ok(ctx)
}
```

- api_create.go > func (h *Handler) ApiCreateForm(ctx *context.Context)
``` shell
# 
func (h *Handler) ApiCreateForm(ctx *context.Context) {

	var (
		params           = guard.GetShowNewFormParam(ctx)
		prefix, paramStr = params.Prefix, params.Param.GetRouteParamStr()
		panel            = h.table(prefix, ctx)
		formInfo         = panel.GetNewFormInfo()
		infoUrl          = h.routePathWithPrefix("api_info", prefix) + paramStr
		newUrl           = h.routePathWithPrefix("api_new", prefix)
		referer          = ctx.Referer()
		f                = panel.GetActualNewForm()
	)

	if referer != "" && !isInfoUrl(referer) && !isNewUrl(referer, ctx.Query(constant.PrefixKey)) {
		infoUrl = referer
	}

	response.OkWithData(ctx, map[string]interface{}{
		"panel": formInfo,
		"urls": map[string]string{
			"info": infoUrl,
			"new":  newUrl,
		},
		"pk":     panel.GetPrimaryKey().Name,
		"header": f.HeaderHtml,
		"footer": f.FooterHtml,
		"prefix": h.config.PrefixFixSlash(),
		"token":  h.authSrv().AddToken(),
		"operation_footer": formFooter("new", f.IsHideContinueEditCheckBox, f.IsHideContinueNewCheckBox,
			f.IsHideResetButton, f.FormNewBtnWord),
	})
}
```


- api_detail.go > func (h *Handler) ApiDetail(ctx *context.Context)
``` shell
# 
func (h *Handler) ApiDetail(ctx *context.Context) {
	prefix := ctx.Query(constant.PrefixKey)
	id := ctx.Query(constant.DetailPKKey)
	panel := h.table(prefix, ctx)
	user := auth.Auth(ctx)

	newPanel := panel.Copy()

	formModel := newPanel.GetForm()

	var fieldList types.FieldList

	if len(panel.GetDetail().FieldList) == 0 {
		fieldList = panel.GetInfo().FieldList
	} else {
		fieldList = panel.GetDetail().FieldList
	}

	formModel.FieldList = make([]types.FormField, len(fieldList))

	for i, field := range fieldList {
		formModel.FieldList[i] = types.FormField{
			Field:        field.Field,
			FieldClass:   field.Field,
			TypeName:     field.TypeName,
			Head:         field.Head,
			Hide:         field.Hide,
			Joins:        field.Joins,
			FormType:     form.Default,
			FieldDisplay: field.FieldDisplay,
		}
	}

	param := parameter.GetParam(ctx.Request.URL,
		panel.GetInfo().DefaultPageSize,
		panel.GetInfo().SortField,
		panel.GetInfo().GetSort())

	paramStr := param.DeleteDetailPk().GetRouteParamStr()

	editUrl := modules.AorEmpty(!panel.GetInfo().IsHideEditButton, h.routePathWithPrefix("show_edit", prefix)+paramStr+
		"&"+constant.EditPKKey+"="+ctx.Query(constant.DetailPKKey))
	deleteUrl := modules.AorEmpty(!panel.GetInfo().IsHideDeleteButton, h.routePathWithPrefix("delete", prefix)+paramStr)
	infoUrl := h.routePathWithPrefix("info", prefix) + paramStr

	editUrl = user.GetCheckPermissionByUrlMethod(editUrl, h.route("show_edit").Method())
	deleteUrl = user.GetCheckPermissionByUrlMethod(deleteUrl, h.route("delete").Method())

	deleteJs := ""

	if deleteUrl != "" {
		deleteJs = fmt.Sprintf(`<script>
function DeletePost(id) {
	swal({
			title: '%s',
			type: "warning",
			showCancelButton: true,
			confirmButtonColor: "#DD6B55",
			confirmButtonText: '%s',
			closeOnConfirm: false,
			cancelButtonText: '%s',
		},
		function () {
			$.ajax({
				method: 'post',
				url: '%s',
				data: {
					id: id
				},
				success: function (data) {
					if (typeof (data) === "string") {
						data = JSON.parse(data);
					}
					if (data.code === 200) {
						location.href = '%s'
					} else {
						swal(data.msg, '', 'error');
					}
				}
			});
		});
}

$('.delete-btn').on('click', function (event) {
	DeletePost(%s)
});

</script>`, language.Get("are you sure to delete"), language.Get("yes"), language.Get("cancel"), deleteUrl, infoUrl, id)
	}

	desc := panel.GetDetail().Description

	if desc == "" {
		desc = panel.GetInfo().Description + language.Get("Detail")
	}

	formInfo, err := newPanel.GetDataWithId(param.WithPKs(id))

	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.OkWithData(ctx, map[string]interface{}{
		"panel":    formInfo,
		"previous": infoUrl,
		"footer":   deleteJs,
		"prefix":   h.config.PrefixFixSlash(),
	})
}

```


- api_list.go > func (h *Handler) ApiList(ctx *context.Context)
``` shell
# 
func (h *Handler) ApiList(ctx *context.Context) {
	prefix := ctx.Query(constant.PrefixKey)

	panel := h.table(prefix, ctx)

	params := parameter.GetParam(ctx.Request.URL, panel.GetInfo().DefaultPageSize, panel.GetInfo().SortField,
		panel.GetInfo().GetSort())

	panel, panelInfo, urls, err := h.showTableData(ctx, prefix, params, panel, "api_")
	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.OkWithData(ctx, map[string]interface{}{
		"panel":  panelInfo,
		"footer": panelInfo.Paginator.GetContent() + panel.GetInfo().FooterHtml,
		"header": aDataTable().GetDataTableHeader() + panel.GetInfo().HeaderHtml,
		"prefix": h.config.PrefixFixSlash(),
		"urls": map[string]string{
			"edit":   urls[0],
			"new":    urls[1],
			"delete": urls[2],
			"export": urls[3],
			"detail": urls[4],
			"info":   urls[5],
			"update": urls[6],
		},
	})
}
```


- api_update.go > func (h *Handler) ApiUpdate(ctx *context.Context)
``` shell
# 
func (h *Handler) ApiUpdate(ctx *context.Context) {
	param := guard.GetEditFormParam(ctx)

	if len(param.MultiForm.File) > 0 {
		err := file.GetFileEngine(h.config.FileUploadEngine.Name).Upload(param.MultiForm)
		if err != nil {
			response.Error(ctx, err.Error())
			return
		}
	}

	for i := 0; i < len(param.Panel.GetForm().FieldList); i++ {
		if param.Panel.GetForm().FieldList[i].FormType == form.File &&
			len(param.MultiForm.File[param.Panel.GetForm().FieldList[i].Field]) == 0 &&
			param.MultiForm.Value[param.Panel.GetForm().FieldList[i].Field+"__delete_flag"][0] != "1" {
			delete(param.MultiForm.Value, param.Panel.GetForm().FieldList[i].Field)
		}
	}

	err := param.Panel.UpdateData(param.Value())
	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.Ok(ctx)
}

```

- api_update.go > func (h *Handler) ApiUpdateForm(ctx *context.Context)
``` shell
# 
func (h *Handler) ApiUpdateForm(ctx *context.Context) {
	params := guard.GetShowFormParam(ctx)

	prefix, param := params.Prefix, params.Param

	panel := h.table(prefix, ctx)

	user := auth.Auth(ctx)

	paramStr := param.GetRouteParamStr()

	newUrl := modules.AorEmpty(panel.GetCanAdd(), h.routePathWithPrefix("api_show_new", prefix)+paramStr)
	footerKind := "edit"
	if newUrl == "" || !user.CheckPermissionByUrlMethod(newUrl, h.route("api_show_new").Method(), url.Values{}) {
		footerKind = "edit_only"
	}

	formInfo, err := panel.GetDataWithId(param)

	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	infoUrl := h.routePathWithPrefix("api_info", prefix) + param.DeleteField(constant.EditPKKey).GetRouteParamStr()
	editUrl := h.routePathWithPrefix("api_edit", prefix)

	f := panel.GetForm()

	response.OkWithData(ctx, map[string]interface{}{
		"panel": formInfo,
		"urls": map[string]string{
			"info": infoUrl,
			"edit": editUrl,
		},
		"pk":     panel.GetPrimaryKey().Name,
		"header": f.HeaderHtml,
		"footer": f.FooterHtml,
		"prefix": h.config.PrefixFixSlash(),
		"token":  h.authSrv().AddToken(),
		"operation_footer": formFooter(footerKind, f.IsHideContinueEditCheckBox, f.IsHideContinueNewCheckBox,
			f.IsHideResetButton, f.FormEditBtnWord),
	})
}
```




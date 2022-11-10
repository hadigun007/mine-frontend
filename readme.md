## Controller 
<!-- https://github.com/GoAdminGroup/go-admin@v1.2.23/plugins/admin/controller -->


- [ApiCreate](#api_creatego--apicreate)
- [ApiCreateForm](#api_creatego--apicreateform)
- [ApiDetail](#api_detailgo--apidetail)
- [ApiList](#api_listgo--apilist)
- [ApiUpdate](#api_updatego--apiupdate)
- [ApiUpdateForm](#api_updatego--apiupdateform)
- [Auth](#authgo--auth)
- [Logout](#authgo--logout)
- [ShowLogin](#authgo--showlogin)
- [TestInfoUrl]
- [TestIsNewUrl]
- [Delete](#deletego--delete)
- [ShowDetail](#detailgo--detail)
- [ShowForm](#editgo--showform)
- [EditForm](#editgo--editform)
- [GlobalDetaultHandler](#handlergo--globaldefenderhandler)
- [ShowInstall](#installgo--showinstall)
- [CheckDatabase](#installgo--checkdatabase)
- [ShowMenu] 
- [ShowNewMenu] 
- [ShowEditMenu] 
- [DeleteMenu] 
- [EditMenu] 
- [NewMenu] 
- [MenuOrder] 
- [ShowNewForm] 
- [NewForm] 
- [Operation] 
- [RecordOperationLog]
- [GetPluginsPageJS] 
- [Plugins] 
- [PluginStore] 
- [PluginDetail] 
- [GetPluginBoxParamFromPlug]
- [PluginDownload] 
- [ServerLogin] 
- [ShowInfo] 
- [Assets] 
- [Export] 
- [SystemInfo] 
- [Update] 


-----------------------------------
### Controller API

##### api_create.go > ApiCreate
``` shell
# Learning
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

##### api_create.go > ApiCreateForm
``` shell
# Learning
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

##### api_detail.go > ApiDetail
``` shell
# Learning
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

##### api_list.go > ApiList
``` shell
# Learning
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

##### api_update.go > ApiUpdate
``` shell
# Learning
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

##### api_update.go > ApiUpdateForm
``` shell
# Learning
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

##### auth.go > Auth 
``` shell
# Auth check input passsword and username for authentication.
func (h *Handler) Auth(ctx *context.Context) {

	var (
		user     models.UserModel
		ok       bool
		errMsg   = "fail"
		s, exist = h.services.GetOrNot(auth.ServiceKey)
	)

	if capDriver, ok := h.captchaConfig["driver"]; ok {
		capt, ok := captcha.Get(capDriver)

		if ok {
			if !capt.Validate(ctx.FormValue("token")) {
				response.BadRequest(ctx, "wrong captcha")
				return
			}
		}
	}

	if !exist {
		password := ctx.FormValue("password")
		username := ctx.FormValue("username")

		if password == "" || username == "" {
			response.BadRequest(ctx, "wrong password or username")
			return
		}
		user, ok = auth.Check(password, username, h.conn)
	} else {
		user, ok, errMsg = auth.GetService(s).P(ctx)
	}

	if !ok {
		response.BadRequest(ctx, errMsg)
		return
	}

	err := auth.SetCookie(ctx, user, h.conn)

	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	if ref := ctx.Referer(); ref != "" {
		if u, err := url.Parse(ref); err == nil {
			v := u.Query()
			if r := v.Get("ref"); r != "" {
				rr, _ := url.QueryUnescape(r)
				response.OkWithData(ctx, map[string]interface{}{
					"url": rr,
				})
				return
			}
		}
	}

	response.OkWithData(ctx, map[string]interface{}{
		"url": h.config.GetIndexURL(),
	})
}
```

##### auth.go > Logout
``` shell
# Logout delete the cookie
func (h *Handler) Logout(ctx *context.Context) {
	err := auth.DelCookie(ctx, db.GetConnection(h.services))
	if err != nil {
		logger.Error("logout error", err)
	}
	ctx.AddHeader("Location", h.config.Url(config.GetLoginUrl()))
	ctx.SetStatusCode(302)
}
```

##### auth.go > ShowLogin
``` shell
# show the login page
// ShowLogin show the login page.
func (h *Handler) ShowLogin(ctx *context.Context) {

	tmpl, name := template.GetComp("login").GetTemplate()
	buf := new(bytes.Buffer)
	if err := tmpl.ExecuteTemplate(buf, name, struct {
		UrlPrefix string
		Title     string
		Logo      template2.HTML
		CdnUrl    string
		System    types.SystemInfo
	}{
		UrlPrefix: h.config.AssertPrefix(),
		Title:     h.config.LoginTitle,
		Logo:      h.config.LoginLogo,
		System: types.SystemInfo{
			Version: system.Version(),
		},
		CdnUrl: h.config.AssetUrl,
	}); err == nil {
		ctx.HTML(http.StatusOK, buf.String())
	} else {
		logger.Error(err)
		ctx.HTML(http.StatusOK, "parse template error (；′⌒`)")
	}
}
```

##### delete.go > Delete 
``` shell
# Delete delete the row from database
func (h *Handler) Delete(ctx *context.Context) {

	param := guard.GetDeleteParam(ctx)

	//token := ctx.FormValue("_t")
	//
	//if !auth.TokenHelper.CheckToken(token) {
	//	ctx.SetStatusCode(http.StatusBadRequest)
	//	ctx.WriteString(`{"code":400, "msg":"delete fail"}`)
	//	return
	//}

	if err := h.table(param.Prefix, ctx).DeleteData(param.Id); err != nil {
		logger.Error(err)
		response.Error(ctx, "delete fail")
		return
	}

	response.OkWithData(ctx, map[string]interface{}{
		"token": h.authSrv().AddToken(),
	})
}
```

##### detail.go > Detail
``` shell
# ..
func (h *Handler) ShowDetail(ctx *context.Context) {

	var (
		prefix    = ctx.Query(constant.PrefixKey)
		id        = ctx.Query(constant.DetailPKKey)
		panel     = h.table(prefix, ctx)
		user      = auth.Auth(ctx)
		newPanel  = panel.Copy()
		detail    = panel.GetDetail()
		info      = panel.GetInfo()
		formModel = newPanel.GetForm()
		fieldList = make(types.FieldList, 0)
	)

	if len(detail.FieldList) == 0 {
		fieldList = info.FieldList
	} else {
		fieldList = detail.FieldList
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

	if detail.Table != "" {
		formModel.Table = detail.Table
	} else {
		formModel.Table = info.Table
	}

	param := parameter.GetParam(ctx.Request.URL,
		info.DefaultPageSize,
		info.SortField,
		info.GetSort())

	paramStr := param.DeleteDetailPk().GetRouteParamStr()

	editUrl := modules.AorEmpty(!info.IsHideEditButton, h.routePathWithPrefix("show_edit", prefix)+paramStr+
		"&"+constant.EditPKKey+"="+ctx.Query(constant.DetailPKKey))
	deleteUrl := modules.AorEmpty(!info.IsHideDeleteButton, h.routePathWithPrefix("delete", prefix)+paramStr)
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

</script>`, language.Get("are you sure to delete"), language.Get("yes"),
			language.Get("cancel"), deleteUrl, infoUrl, id)
	}

	title := ""
	desc := ""

	isNotIframe := ctx.Query(constant.IframeKey) != "true"

	if isNotIframe {
		title = detail.Title

		if title == "" {
			title = info.Title + language.Get("Detail")
		}

		desc = detail.Description

		if desc == "" {
			desc = info.Description + language.Get("Detail")
		}
	}

	formInfo, err := newPanel.GetDataWithId(param.WithPKs(id))

	if err != nil {
		h.HTML(ctx, user, template.WarningPanelWithDescAndTitle(err.Error(), desc, title),
			template.ExecuteOptions{Animation: param.Animation})
		return
	}

	h.HTML(ctx, user, types.Panel{
		Content: detailContent(aForm().
			SetTitle(template.HTML(title)).
			SetContent(formInfo.FieldList).
			SetHeader(detail.HeaderHtml).
			SetFooter(template.HTML(deleteJs)+detail.FooterHtml).
			SetHiddenFields(map[string]string{
				form2.PreviousKey: infoUrl,
			}).
			SetPrefix(h.config.PrefixFixSlash()), editUrl, deleteUrl, !isNotIframe),
		Description: template.HTML(desc),
		Title:       template.HTML(title),
	}, template.ExecuteOptions{Animation: param.Animation})
}
```

##### edit.go > ShowForm
``` shell
# ShowFrom show form page
func (h *Handler) ShowForm(ctx *context.Context) {
	param := guard.GetShowFormParam(ctx)
	h.showForm(ctx, "", param.Prefix, param.Param, false)
}

```

##### edit.go > EditForm
``` shell
# ...
func (h *Handler) EditForm(ctx *context.Context) {

	param := guard.GetEditFormParam(ctx)

	if len(param.MultiForm.File) > 0 {
		err := file.GetFileEngine(h.config.FileUploadEngine.Name).Upload(param.MultiForm)
		if err != nil {
			logger.Error("get file engine error: ", err)
			if ctx.WantJSON() {
				response.Error(ctx, err.Error())
			} else {
				h.showForm(ctx, aAlert().Warning(err.Error()), param.Prefix, param.Param, true)
			}
			return
		}
	}

	formPanel := param.Panel.GetForm()

	for i := 0; i < len(formPanel.FieldList); i++ {
		if formPanel.FieldList[i].FormType == form.File &&
			len(param.MultiForm.File[formPanel.FieldList[i].Field]) == 0 &&
			len(param.MultiForm.Value[formPanel.FieldList[i].Field+"__delete_flag"]) > 0 &&
			param.MultiForm.Value[formPanel.FieldList[i].Field+"__delete_flag"][0] != "1" {
			param.MultiForm.Value[formPanel.FieldList[i].Field] = []string{""}
		}
		if formPanel.FieldList[i].FormType == form.File &&
			len(param.MultiForm.Value[formPanel.FieldList[i].Field+"__change_flag"]) > 0 &&
			param.MultiForm.Value[formPanel.FieldList[i].Field+"__change_flag"][0] != "1" {
			delete(param.MultiForm.Value, formPanel.FieldList[i].Field)
		}
	}

	err := param.Panel.UpdateData(param.Value())
	if err != nil {
		logger.Error("update data error: ", err)
		if ctx.WantJSON() {
			response.Error(ctx, err.Error(), map[string]interface{}{
				"token": h.authSrv().AddToken(),
			})
		} else {
			h.showForm(ctx, aAlert().Warning(err.Error()), param.Prefix, param.Param, true)
		}
		return
	}

	if formPanel.Responder != nil {
		formPanel.Responder(ctx)
		return
	}

	if ctx.WantJSON() && !param.IsIframe {
		response.OkWithData(ctx, map[string]interface{}{
			"url":   param.PreviousPath,
			"token": h.authSrv().AddToken(),
		})
		return
	}

	if !param.FromList {

		if isNewUrl(param.PreviousPath, param.Prefix) {
			h.showNewForm(ctx, param.Alert, param.Prefix, param.Param.DeleteEditPk().GetRouteParamStr(), true)
			return
		}

		if isEditUrl(param.PreviousPath, param.Prefix) {
			h.showForm(ctx, param.Alert, param.Prefix, param.Param, true, false)
			return
		}

		ctx.HTML(http.StatusOK, fmt.Sprintf(`<script>location.href="%s"</script>`, param.PreviousPath))
		ctx.AddHeader(constant.PjaxUrlHeader, param.PreviousPath)
		return
	}

	if param.IsIframe {
		ctx.HTML(http.StatusOK, fmt.Sprintf(`<script>
		swal('%s', '', 'success');
		setTimeout(function(){
			$("#%s", window.parent.document).hide();
			$('.modal-backdrop.fade.in', window.parent.document).hide();
		}, 1000)
</script>`, language.Get("success"), param.IframeID))
		return
	}

	buf := h.showTable(ctx, param.Prefix, param.Param.DeletePK().DeleteEditPk(), nil)

	ctx.HTML(http.StatusOK, buf.String())
	ctx.AddHeader(constant.PjaxUrlHeader, param.PreviousPath)
}
```

##### handler.go > GlobalDefenderHandler
``` shell
# GlobalHandlerDefender is a global error handler of admin plugin.
func (h *Handler) GlobalDeferHandler(ctx *context.Context) {
	fmt.Println("Controller")
	logger.Access(ctx)

	if !h.config.OperationLogOff {
		h.RecordOperationLog(ctx)
	}

	if err := recover(); err != nil {
		logger.Error(err)

		var (
			errMsg string
			ok     bool
			e      error
		)

		if errMsg, ok = err.(string); !ok {
			if e, ok = err.(error); ok {
				errMsg = e.Error()
			}
		}

		if errMsg == "" {
			errMsg = "system error"
		}

		if ctx.WantJSON() {
			response.Error(ctx, errMsg)
			return
		}

		if ok, _ = regexp.MatchString("/edit(.*)", ctx.Path()); ok {
			h.setFormWithReturnErrMessage(ctx, errMsg, "edit")
			return
		}
		if ok, _ = regexp.MatchString("/new(.*)", ctx.Path()); ok {
			h.setFormWithReturnErrMessage(ctx, errMsg, "new")
			return
		}

		h.HTML(ctx, auth.Auth(ctx), template.WarningPanelWithDescAndTitle(errMsg, errors.Msg, errors.Msg))
	}
}

```

##### install.go > ShowInstall 
``` shell
# ShowInstall show install page
func (h *Handler) ShowInstall(ctx *context.Context) {

	buffer := new(bytes.Buffer)
	//template.GetInstallPage(buffer)

	//rs, _ := mysql.Query("show tables;")
	//fmt.Println(rs[0]["Tables_in_godmin"])

	//rs2, _ := mysql.Query("show columns from users")
	//fmt.Println(rs2[0]["Field"])

	ctx.HTML(http.StatusOK, buffer.String())
}
```

##### install.go > CheckDatabase
``` shell 
# CheckDatabase check the database connection
func (h *Handler) CheckDatabase(ctx *context.Context) {

	ip := ctx.FormValue("h")
	port := ctx.FormValue("po")
	username := ctx.FormValue("u")
	password := ctx.FormValue("pa")
	databaseName := ctx.FormValue("db")

	SqlDB, err := sql.Open("mysql", username+":"+password+"@tcp("+ip+":"+port+")/"+databaseName+"?charset=utf8mb4")
	if SqlDB != nil {
		if SqlDB.Ping() != nil {
			response.Error(ctx, "请检查参数是否设置正确")
			return
		}
	}

	defer func() {
		_ = SqlDB.Close()
	}()

	if err != nil {
		response.Error(ctx, "请检查参数是否设置正确")
		return

	}

	//db.InitDB(username, password, port, ip, databaseName, 100, 100)

	tables := make([]map[string]interface{}, 0)

	list := "["

	for i := 0; i < len(tables); i++ {
		if i != len(tables)-1 {
			list += `"` + tables[i]["Tables_in_godmin"].(string) + `",`
		} else {
			list += `"` + tables[i]["Tables_in_godmin"].(string) + `"`
		}
	}
	list += "]"

	response.OkWithData(ctx, map[string]interface{}{
		"list": list,
	})
}
```
##### menu.go > 